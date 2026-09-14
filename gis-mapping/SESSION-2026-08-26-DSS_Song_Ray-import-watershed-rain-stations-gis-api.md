# Session 2026-08-26 — Import watershed boundary + rain stations, expose as GIS API

## 1. Requirement recap

User dropped a new GIS data folder (`C:\Users\Admin\Downloads\Data (1)\Data\gis`) and asked to
inspect what it contained, then implement an ingestion pipeline and API endpoints so the frontend
map could display it: *"đọc data trong folder này, xem có data gì được cung cấp cho dự án này, thì
triển khai import rồi viết API trả cho FE."*

## 2. How it was implemented + docs used

**Discovery.** No GDAL tooling (`ogrinfo`, `gdalinfo`, `geopandas`) was available locally — the
bundled `docs/DULIEU_BDHC_VUNGTAU/ogrinfo.exe` failed to run (missing a shared library). An Explore
subagent instead parsed the raw `.shp`/`.dbf`/`.prj`/`.tfw` binary and text formats directly to
extract geometry type, field schema, CRS WKT, and record counts, without GDAL. This finding — "no
usable GDAL binary locally" — is reported by that subagent and was not independently re-verified
in the main session; treat it as inferred, not confirmed firsthand.

Data found, three shapefiles (raster DEM/hillshade files were already served by the existing
TiTiler pipeline, so out of scope):
- `luuvuc_SongRay.shp` — 1 polygon, Sông Ray watershed boundary, 696.48 km², EPSG:32648.
- `songray.shp` — 1 polyline, river centerline (KMZ-derived), EPSG:32648.
- `trammua.shp` — 14 points, rain gauge stations, names already UTF-8, EPSG:32648.

Key CRS distinction: the `.prj` WKT read `WGS_1984_UTM_Zone_48N`, central meridian 105°E — standard
EPSG:32648. This is *different* from the project's existing legacy BDHC shapefiles, which use a
custom Transverse Mercator (central meridian 107.75°E) plus TCVN3 text encoding. Getting this wrong
would silently reproject the new data to the wrong location.

**Approach.** Followed the existing GIS-ingestion pattern in this repo exactly (NestJS + TypeORM +
PostGIS, `ST_GeomFromGeoJSON` raw inserts, TRUNCATE-then-reload idempotent scripts, JWT-guarded
`GET /gis/*` GeoJSON endpoints): see `apps/api/src/gis/`, `apps/api/scripts/gis-ingest/`.

Rather than writing a new shapefile→GeoJSON converter, `shp-to-geojson.ts`'s `convertToGeoJson()`
was generalized to accept optional `sourceDir`/`sourceCrs`/`encoding` overrides, defaulting to the
old hardcoded BDHC values — so all 4 pre-existing ingest scripts keep working unchanged. Rejected
alternative: duplicating the converter for the new dataset — rejected per DRY, since the only
actual difference was 3 parameters, not the Docker/ogr2ogr mechanism itself.

The 3 shapefiles were copied into the repo at `docs/GIS_SongRay/` (mirroring the existing
`docs/DULIEU_BDHC_VUNGTAU/` convention) for a stable, versioned source path. ArcGIS `.lock` files
that shipped alongside them were discarded (transient, not needed).

**Schema decisions:**
- New table `watershed_boundaries` (MultiPolygon + GiST) — no existing table fit a named/measured
  basin polygon.
- New table `rain_stations` (Point + GiST) — kept separate from the existing `poi_points` table
  because rain gauges are a distinct semantic layer the frontend toggles independently, the same
  way monitoring `stations` already gets its own dedicated endpoint instead of being folded into
  `poi_points`.
- The river centerline (1 feature) got **no** new table — inserted into the existing
  `hydrology_features` table with `feature_type='river'`, since that table is already the generic
  home for river/lake/reservoir geometries; a whole table for one line feature would violate YAGNI.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/api/scripts/gis-ingest/shp-to-geojson.ts` | shapefile→GeoJSON via Docker `ogr2ogr` | Hardcoded `SOURCE_DIR`/`SOURCE_CRS`/`LATIN1` for BDHC only | Accepts optional `ConvertOptions` (`sourceDir`, `sourceCrs`, `encoding`), defaults preserved |
| `apps/api/scripts/gis-ingest/ingest-watershed.ts` | new ingest script | did not exist | Loads `luuvuc_SongRay.shp` → `watershed_boundaries`, `songray.shp` → `hydrology_features` |
| `apps/api/scripts/gis-ingest/ingest-rain-stations.ts` | new ingest script | did not exist | Loads `trammua.shp` → `rain_stations` |
| `apps/api/src/database/migrations/20260826090000-InitWatershedRain.ts` | schema migration | did not exist | Creates `watershed_boundaries` + `rain_stations` tables with GiST indexes |
| `apps/api/src/gis/entities/watershed-boundary.entity.ts`, `rain-station.entity.ts` | TypeORM entities | did not exist | Map new tables to `WatershedBoundary`/`RainStation` |
| `apps/api/src/gis/gis.service.ts` | GIS query service | 3 existing `get*` methods | + `getWatershedBoundary()`, `getRainStations()` |
| `apps/api/src/gis/gis.controller.ts` | GIS HTTP routes | 4 existing `@Get` routes | + `GET /gis/watershed`, `GET /gis/rain-stations` |
| `apps/api/src/gis/gis.module.ts` | module wiring | 5 repos registered | + `WatershedBoundary`, `RainStation` repos |
| `apps/api/package.json`, `run-ingest.ps1` | script registry | 6 gis-ingest scripts | + `gis-ingest:watershed`, `gis-ingest:rain-stations`, wired into `run-all` |
| `apps/web/src/services/gisService.ts` | FE API client | 3 getters | + `getWatershedBoundary()`, `getRainStations()` |
| `apps/web/src/lib/tokens.ts` | color tokens | no `watershed` color | + `MAP_COLORS.watershed = "#059669"` |
| `apps/web/src/components/gis/AppMapCanvas.tsx` | map rendering | 3 layers/markers | + watershed line layer, rain-station markers, 2 new `LayerVisibility` fields |
| `apps/web/src/components/gis/LayerControls.tsx` | layer toggle panel | "Mưa" was an inert `PlaceholderLayer` doing nothing | "Mưa" removed from placeholder set; real "Trạm mưa" + "Lưu vực Sông Ray" toggles added, wired to `visibility` |
| `apps/web/src/pages/GisMapPage.tsx`, `DashboardPage.tsx` | page-level default visibility | `LayerVisibility` literal missing 2 keys | Added `watershed: true, rainStations: true` (required by the interface, not optional) |

## 4. Code changes in detail

### 1. Parameterize the shapefile converter — `apps/api/scripts/gis-ingest/shp-to-geojson.ts`

**Before:**
```ts
export function convertToGeoJson(shapefile: string, outDir: string): GeoJsonFeature[] {
  const outFile = join(outDir, `${shapefile}.geojson`);
  execFileSync("docker", [
    "run", "--rm",
    "-v", `${SOURCE_DIR}:/data`,
    "-v", `${outDir.replace(/\\/g, "/")}:/out`,
    GDAL_IMAGE,
    ...
    "-oo", "ENCODING=LATIN1",
    "-s_srs", SOURCE_CRS,
    "-t_srs", "EPSG:4326",
  ]);
```

**After:**
```ts
export interface ConvertOptions {
  sourceDir?: string;
  sourceCrs?: string;
  encoding?: string;
}

export function convertToGeoJson(
  shapefile: string,
  outDir: string,
  options: ConvertOptions = {},
): GeoJsonFeature[] {
  const sourceDir = options.sourceDir ?? SOURCE_DIR;
  const sourceCrs = options.sourceCrs ?? SOURCE_CRS;
  const encoding = options.encoding ?? "LATIN1";
  const outFile = join(outDir, `${shapefile}.geojson`);
  execFileSync("docker", [
    "run", "--rm",
    "-v", `${sourceDir}:/data`,
    "-v", `${outDir.replace(/\\/g, "/")}:/out`,
    GDAL_IMAGE,
    ...
    "-oo", `ENCODING=${encoding}`,
    "-s_srs", sourceCrs,
    "-t_srs", "EPSG:4326",
  ]);
```

**What changed:** added an optional `ConvertOptions` third parameter; `SOURCE_DIR`, `SOURCE_CRS`,
`"LATIN1"` are now fallback defaults instead of hardcoded values.
**Why:** the new dataset needs a different source directory, a different CRS (EPSG:32648 vs. the
legacy custom TM), and UTF-8 instead of LATIN1/TCVN3 — without this, either the 4 existing BDHC
scripts would need editing (risking breakage) or a duplicate converter would be needed (DRY
violation).
**How it behaves now:** the 4 existing callers, which pass no third argument, get byte-identical
behavior; the 2 new scripts pass `{ sourceDir, sourceCrs: "EPSG:32648", encoding: "UTF-8" }` and
get correctly-reprojected, correctly-decoded output.

### 2. New ingest scripts — `ingest-watershed.ts`, `ingest-rain-stations.ts`

**Before:** (did not exist)

**After:** both scripts follow the identical shape: `dataSource.initialize()` → `TRUNCATE` the
target table (idempotent re-run) → `convertToGeoJson(shapefile, tmpDir, options)` → loop features,
`INSERT ... ST_GeomFromGeoJSON($n)` wrapped in `ST_SetSRID(..., 4326)` (and `ST_Multi(...)` for the
polygon table, since the column type is `MultiPolygon` but a single-feature shapefile parses as a
plain `Polygon`). `ingest-watershed.ts` additionally deletes/reinserts one `hydrology_features` row
tagged `name = 'Song Ray (centerline)'` for the river polyline.

**What changed:** net-new files.
**Why:** gives the two new datasets their own idempotent, re-runnable ingest path consistent with
the other 5 scripts in the folder, wired into `run-ingest.ps1`.
**How it behaves now:** `pnpm gis-ingest:watershed` and `pnpm gis-ingest:rain-stations` populate
`watershed_boundaries`/`rain_stations`/`hydrology_features` from the shapefiles in
`docs/GIS_SongRay/`; re-running either script is safe (TRUNCATE/DELETE first).

### 3. New tables — `apps/api/src/database/migrations/20260826090000-InitWatershedRain.ts`

**Before:** (did not exist)

**After:**
```sql
CREATE TABLE "watershed_boundaries" (
  "id" uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  "name" varchar,
  "area_km2" numeric,
  "geom" geometry(MultiPolygon, 4326) NOT NULL
);
CREATE INDEX "IDX_watershed_boundaries_geom" ON "watershed_boundaries" USING GiST ("geom");

CREATE TABLE "rain_stations" (
  "id" uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  "name" varchar,
  "geom" geometry(Point, 4326) NOT NULL
);
CREATE INDEX "IDX_rain_stations_geom" ON "rain_stations" USING GiST ("geom");
```

**What changed:** two new tables, both with a spatial GiST index, matching the style of the
existing `20260824041041-InitGisBasemap.ts` migration.
**Why:** persistent storage for the new watershed polygon and rain-gauge points.
**How it behaves now:** `down()` drops both tables in reverse order; migration has not been run
against a live database in this session (see section 7).

### 4. New API endpoints — `apps/api/src/gis/gis.controller.ts`, `gis.service.ts`, `gis.module.ts`

**Before:** `GisController` exposed `admin-boundaries`, `hydrology`, `roads`, `stations`,
`reservoir-info`; `GisService` had matching methods; `GisModule` registered 5 repos.

**After (controller):**
```ts
@Get("watershed")
@ApiOperation({ summary: "Fetch the Song Ray watershed/basin boundary polygon as GeoJSON" })
@ApiResponse({ status: 200, description: "GeoJSON FeatureCollection" })
getWatershedBoundary() {
  return this.gisService.getWatershedBoundary();
}

@Get("rain-stations")
@ApiOperation({ summary: "Fetch rain gauge stations (trạm mưa) as GeoJSON points" })
@ApiResponse({ status: 200, description: "GeoJSON FeatureCollection" })
getRainStations() {
  return this.gisService.getRainStations();
}
```

**After (service):**
```ts
async getWatershedBoundary(): Promise<GeoJsonFeatureCollection> {
  const rows = await this.watershedRepo
    .createQueryBuilder("w")
    .select(["w.id", "w.name", "w.areaKm2"])
    .addSelect("ST_AsGeoJSON(w.geom)", "geom_json")
    .getRawMany();
  return { type: "FeatureCollection", features: rows.map((r) => rowToFeature(r, "w")) };
}

async getRainStations(): Promise<GeoJsonFeatureCollection> {
  const rows = await this.rainStationRepo
    .createQueryBuilder("rs")
    .select(["rs.id", "rs.name"])
    .addSelect("ST_AsGeoJSON(rs.geom)", "geom_json")
    .getRawMany();
  return { type: "FeatureCollection", features: rows.map((r) => rowToFeature(r, "rs")) };
}
```

`GisModule`'s `TypeOrmModule.forFeature([...])` array gained `WatershedBoundary, RainStation` and
both are injected into `GisService`'s constructor.

**What changed:** two new routes, two new service methods reusing the shared `rowToFeature` helper
already used by the other 4 endpoints, two new repo injections.
**Why:** matches the existing pattern exactly so the frontend gets consistent GeoJSON
FeatureCollection shapes across all `/gis/*` routes, and both are JWT-guarded like the rest.
**How it behaves now:** `GET /gis/watershed` returns 1 MultiPolygon feature with `name`/`areaKm2`
properties; `GET /gis/rain-stations` returns 14 Point features with `name`.

### 5. Frontend map layer + real "Trạm mưa" toggle — `AppMapCanvas.tsx`, `LayerControls.tsx`, `gisService.ts`, `tokens.ts`

**Before (`LayerControls.tsx`):** `"Mưa"` was one of six entries in a `PlaceholderLayer` union type
whose checkbox flipped local `placeholderOn` state and did nothing else — a dead UI element.
```ts
type PlaceholderLayer = "rain" | "floodMap" | "riskZone" | "scenarioResult" | "roads" | "satellite";
...
<label className="layer-controls-item">
  <input type="checkbox" checked={placeholderOn.rain} onChange={() => togglePlaceholder("rain")} />
  <RainIcon />
  <span className="layer-controls-item-name">Mưa</span>
</label>
```

**After:** `"rain"` removed from `PlaceholderLayer`; two new toggles added near the top of the layer
list, wired to the real `visibility` prop (not local placeholder state):
```tsx
<label className="layer-controls-item">
  <input type="checkbox" checked={visibility.watershed} onChange={() => toggle("watershed")} />
  <BoundaryIcon />
  <span className="layer-controls-item-name">Lưu vực Sông Ray</span>
</label>

<label className="layer-controls-item">
  <input type="checkbox" checked={visibility.rainStations} onChange={() => toggle("rainStations")} />
  <RainIcon />
  <span className="layer-controls-item-name">Trạm mưa</span>
</label>
```

**In `AppMapCanvas.tsx`:** `LayerVisibility` gained `watershed: boolean` and
`rainStations: boolean`; a new `rainStationMarkersRef` array holds MapLibre `Marker`s (parallel to
the existing `stationMarkersRef` loop); `loadLayers()`'s `Promise.all([...])` grew from 3 to 5
calls, adding `getWatershedBoundary(token)` and `getRainStations(token)`; a new `watershed` GeoJSON
source + `watershed-line` line layer was added (`paint: { "line-color": MAP_COLORS.watershed,
"line-width": 2 }`); the visibility-effect hook grew `setLayerVisible("watershed-line", ...)` and a
marker-display loop for `rainStationMarkersRef`.

**What changed:** an inert placeholder checkbox was deleted, two real toggles were added elsewhere
in the list, `MAP_COLORS.watershed = "#059669"` was added to `tokens.ts`, and `gisService.ts` gained
two thin `getJson()` wrappers matching the existing 3.
**Why:** the user asked specifically for an FE-facing API, which meant the FE also had to consume
it — this closes the loop from raw shapefile to a rendered map layer.
**How it behaves now:** toggling "Lưu vực Sông Ray" shows/hides the green watershed outline;
toggling "Trạm mưa" shows/hides 14 markers with name popups, both driven by the real backend data
instead of doing nothing.

### 6. Required-field ripple in page-level defaults — `GisMapPage.tsx`, `DashboardPage.tsx`

**Before:**
```ts
const DEFAULT_VISIBILITY: LayerVisibility = { adminBoundaries: true, hydrology: true, stations: true, terrain: true };
```

**After:**
```ts
const DEFAULT_VISIBILITY: LayerVisibility = {
  adminBoundaries: true,
  hydrology: true,
  stations: true,
  terrain: true,
  watershed: true,
  rainStations: true,
};
```
(identical shape change in `DashboardPage.tsx`'s `DASHBOARD_MAP_VISIBILITY`).

**What changed:** two required fields added to two object literals.
**Why:** `LayerVisibility` is a non-optional interface, so adding `watershed`/`rainStations` to it
made these two literals fail to type-check until updated — see section 7 for how they were found.
**How it behaves now:** both pages default to all layers, including the two new ones, visible.

## 5. How to find this again

- Grep `watershed_boundaries`, `rain_stations`, `WatershedBoundary`, `RainStation`
- Grep `gis-ingest:watershed`, `gis-ingest:rain-stations` (npm scripts)
- Grep `getWatershedBoundary`, `getRainStations` (service/controller/FE client)
- Grep `LayerVisibility` to find all call sites that must stay in sync with the interface
- Routes: `GET /gis/watershed`, `GET /gis/rain-stations`
- Source shapefiles: `docs/GIS_SongRay/luuvuc_SongRay.shp`, `songray.shp`, `trammua.shp`

## 6. Concepts introduced

- **EPSG:32648** — the UTM Zone 48N projected coordinate system, standard for this longitude band
  in Vietnam. Needed here because the new shapefiles use it, distinct from the legacy BDHC dataset's
  custom Transverse Mercator, so the conversion pipeline had to be parameterized instead of assuming
  one fixed source CRS project-wide.
- **`ST_Multi()`** (PostGIS) — coerces a `Polygon` geometry into a `MultiPolygon`. Needed because the
  watershed table's column type is `MultiPolygon` for future-proofing (a basin could have disjoint
  parts), but the single input feature parses from GeoJSON as a plain `Polygon`.
- **TCVN3 vs UTF-8 Vietnamese text encoding** — TCVN3 is a legacy single-byte Vietnamese encoding
  used in older GIS data; this session's new shapefiles are already UTF-8, so no decoding step was
  needed, unlike the existing BDHC ingest scripts.

## 7. Where it got stuck

- **No local GDAL binary to inspect the shapefiles.** The bundled
  `docs/DULIEU_BDHC_VUNGTAU/ogrinfo.exe` failed with a missing shared library dependency. Worked
  around by having an Explore subagent parse `.shp`/`.dbf`/`.prj`/`.tfw` binary/text formats
  directly rather than through GDAL. *This is reported secondhand from that subagent and was not
  independently re-verified in the main session* — treat the "GDAL unavailable" claim as inferred.
- **CRS mismatch risk.** The new data's `.prj` WKT showed `WGS_1984_UTM_Zone_48N` with
  `Central_Meridian: 105.0` — this had to be manually distinguished from the existing project's
  legacy BDHC CRS (custom TM, central meridian 107.75°E, scale factor 0.9996). Silently reusing the
  old hardcoded `SOURCE_CRS`/`LATIN1` values would have reprojected the watershed/rain data to the
  wrong real-world location without any visible error — the fix was parameterizing
  `convertToGeoJson()` rather than trusting the default.
- **Cross-session file collision.** Another concurrent Claude session ("analyze-raster-data-gaps")
  was implementing OpenMeteo rain-raster and HEC-RAS flood-depth-raster layers touching the same
  files (`AppMapCanvas.tsx`, `LayerControls.tsx`, `gisService.ts`, `gis-raster-layers.ts`). This was
  caught via `SendMessage` coordination mid-session, not via a git conflict — this session claimed
  the vector-layer scope (watershed polygon + rain-gauge points) and asked the other session to
  rename its planned raster "Mưa" toggle to avoid colliding with the `rainStations` naming used
  here, and to hold off on the shared files until this session's edits landed. Evidence for the
  `data/gis/cog/flood_depth_peak_cog.tif` and `rain_latest_cog.tif` untracked files seen in
  `git status` at the start of this write-up session is consistent with that concurrent work having
  since progressed. Lesson for future sessions: check `git status` for unexpected in-flight changes
  before assuming a clean file matches what you last read, since multiple agents can be working the
  same repo concurrently.
- **`LayerVisibility` required-field ripple.** Adding `watershed`/`rainStations` to the
  non-optional `LayerVisibility` interface meant every object literal built against it had to be
  updated or it would fail to compile. Two call sites (`GisMapPage.tsx`, `DashboardPage.tsx`) needed
  the update. The exact mechanism used to find them (a `tsc --noEmit` compile error surfacing the
  two sites, versus a proactive grep for `LayerVisibility =`) is not distinguishable from the
  available git history/diff evidence alone — flagging this as unverified rather than asserting
  either mechanism as fact.

## 8. Verify

```bash
pnpm --filter api exec tsc --noEmit -p tsconfig.json
pnpm --filter web exec tsc --noEmit -p tsconfig.json
```
Both should complete with no errors (confirmed clean in this session).

```bash
pnpm --filter api lint
```
Fails with `ERR_PNPM_RECURSIVE_RUN_NO_SCRIPT` — `apps/api` has no `lint` script defined; not a
regression from this session, noted as a gotcha below.

**Not verified in this session:** the migration (`InitWatershedRain20260826090000`) was not run
against a live database, and neither ingest script (`gis-ingest:watershed`, `gis-ingest:rain-stations`)
was executed end-to-end — no confirmed running Postgres/Docker in this session. The new
`GET /gis/watershed` and `GET /gis/rain-stations` routes have therefore not been hit against real
data. This is a known, explicit gap, not an oversight to gloss over.

## 9. Gotchas

- `apps/api` has no `lint` script — don't assume `pnpm --filter api lint` validates anything; rely
  on `tsc --noEmit` for this package until a lint script is added.
- If another new shapefile source shows up with a different CRS/encoding, always check its `.prj`
  WKT central meridian before assuming it matches either the legacy BDHC CRS or EPSG:32648 — this
  project now has two different projected CRSs in play across ingest scripts.
- The river centerline lives inside `hydrology_features`, not its own table — future edits to
  "rivers" queries need to account for `feature_type='river'` rows that aren't in a dedicated table.
- Two ingest scripts (`ingest-watershed.ts`, `ingest-rain-stations.ts`) both truncate their own
  tables independently; running them out of order is safe, but running only one after schema
  changes could leave the other table stale — always run via `gis-ingest:run-all` when in doubt.
- Multiple concurrent Claude sessions can touch the same shared frontend files
  (`AppMapCanvas.tsx`, `LayerControls.tsx`, `gisService.ts`) — check `git status` before trusting
  that a file matches what was last read.
