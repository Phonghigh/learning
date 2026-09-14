# SESSION-2026-08-17 — Flood raster "giật" khi đổi bước thời gian trên `/gis-map`

**Date:** 2026-08-17

---

## 1. Requirement recap

User's own framing (Vietnamese): "kiểm tra lại toàn bộ flow, đánh giá, phân tích xem xét xem tại sao
nó lại chậm, vì khi tôi demo với phương pháp trước đó là bake png, thì nó rất mượt, không có hiện
tượng render lại toàn bộ. Sau khi chuyển qua dùng solution hiện tại thì mỗi lần nó để render lại, kiểu
giựt 1 cái, tôi nghĩ có khi nào ở step encoder không? vì bake png thì không cần step này, mà step này
lại là nặng nhất, kiểm tra xem phải không và có thể triển khai luôn step encoder màu này luôn không."

User suspected the server-side colormap ("encoder màu") step in titiler was the cause of a visible
stutter/flash every time the flood-raster time step changes on `/gis-map`, and asked to verify and,
if confirmed, remove that step.

## 2. How it was implemented + docs used

Diagnostic-first, same discipline as the earlier same-day tile-server perf session: measure the
user's actual theory before touching code.

- `curl` against the already-running titiler container (`127.0.0.1:8090`), same tile
  (`depth_035.tif`), 3 runs each with vs. without the `colormap=...` query param already used by the
  web app — isolates exactly the cost the user was pointing at.
- Result: no meaningful difference (~50ms both ways; one 130ms outlier on the very first "no
  colormap" run, consistent with a cold request, not a colormap cost). This **disproved** the user's
  theory with a direct measurement, not a guess.
- Re-read the code actually responsible for the swap: `web/src/components/map/useMapContextLayers.ts`,
  the effect that runs when the timeline step changes. Found it removed the old MapLibre layer/source
  *before* adding the new one — a real, structural cause, not a performance tuning issue.
- Cross-checked against the original Pha 4 plan
  (`plans/260817-0900-flood-raster-ingest-system/phase-04-web-db-driven-rendering.md`), which already
  specified the correct behavior under "Prefetch — BẮT BUỘC": *"Giữ layer cũ tới khi tile mới `load`
  xong rồi mới gỡ — nếu không sẽ nháy trắng đúng bằng khoảng 800ms đó."* This requirement was skipped
  during the original Pha 4 implementation (earlier the same day) for simplicity — it never got fixed
  until this session, when the user directly compared it against the old bake-PNG UX.
- Compared old vs. new MapLibre source mechanics: the retired bake-PNG approach used an `image`
  source with `updateImage({url, coordinates})` — a single atomic swap call, no gap. The current
  `raster` source has no equivalent API; MapLibre does not expose a stable `setTiles()`, so replacing
  tiles has always meant removing and re-adding source/layer by hand. That hand-rolled step is what
  needed to be redone correctly (crossfade) rather than removed.

Options considered:
- **Remove the colormap step** (user's original ask) — rejected once measurement showed it wasn't the
  cause; would not have fixed anything and would have removed a feature (dynamic color scales) for no
  benefit.
- **Keep single-slot remove-then-add, just prefetch harder** — rejected; prefetching the next likely
  step (already implemented in Pha 4) reduces *how long* the gap lasts but does not remove the gap
  itself, since the visible layer is still deleted before the replacement exists.
- **Crossfade with 2 ping-pong layer slots** (chosen) — matches what the original Pha 4 plan already
  specified; only fully removes the empty-frame gap because the old layer is never deleted until the
  new one has actually finished loading.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `web/src/components/map/useMapContextLayers.ts` | MapLibre effect that swaps the flood-raster layer on timeline step change | Single fixed layer/source id `"flood-raster"`; step-change effect did `removeLayer` + `removeSource` on it, *then* `addSource` + `addLayer` for the new step — 1 frame with no flood layer at all before the new tiles paint | Two ping-pong slot ids `"flood-raster-0"`/`"flood-raster-1"` tracked via `visibleFloodLayerIdRef` (currently on screen) and `pendingFloodLayerIdRef` (loading); step-change effect adds the new step into the *other* slot and only removes the old slot once MapLibre's `sourcedata`/`isSourceLoaded` fires for the new one — the visible layer is never empty |

## 4. Code changes in detail

### 1. Ping-pong slot refs added — `web/src/components/map/useMapContextLayers.ts`

**Before:** *(no equivalent — single hardcoded id `"flood-raster"` used everywhere)*

**After:**
```ts
// Slot ping-pong cho raster ngập ("flood-raster-0"/"flood-raster-1") -xem
// effect đổi bước bên dưới để hiểu vì sao cần 2 slot thay vì 1 id cố định.
const visibleFloodLayerIdRef = useRef<"flood-raster-0" | "flood-raster-1">("flood-raster-0");
const pendingFloodLayerIdRef = useRef<"flood-raster-0" | "flood-raster-1" | null>(null);
```

**What changed:** two new refs added near the existing `demOpacityRef`/`showRiversRef` refs.
**Why:** the crossfade needs to track which slot id is currently on screen (safe to keep) vs. which
one is mid-load (safe to reuse/discard) without triggering a re-render — a `useState` would cause
unnecessary effect re-runs on every load-progress tick.
**How it behaves now:** every other piece of the file that used to hardcode `"flood-raster"` now
reads `visibleFloodLayerIdRef.current` instead, so the "currently displayed" layer id is a single
source of truth.

### 2. Init effect (`onLoad`) uses the ref instead of a fixed id — `web/src/components/map/useMapContextLayers.ts`

**Before:**
```ts
map.addSource("flood-raster", { type: "raster", tiles: [tileUrl], tileSize: 256 });
map.addLayer({
  id: "flood-raster", type: "raster", source: "flood-raster",
  paint: { "raster-opacity": floodOpacity },
}, "rivers-line");
```

**After:**
```ts
const initialId = visibleFloodLayerIdRef.current; // "flood-raster-0", chưa có gì để crossfade lúc mount
map.addSource(initialId, { type: "raster", tiles: [tileUrl], tileSize: 256, maxzoom: 18 });
map.addLayer({
  id: initialId, type: "raster", source: initialId,
  paint: { "raster-opacity": floodOpacity },
}, "rivers-line");
```

**What changed:** id switched from the string literal to `initialId` (still resolves to
`"flood-raster-0"` on first mount); `maxzoom: 18` also carried over from the separate perf-investigation
fix done earlier the same day.
**Why:** keeps the mount-time layer id consistent with whatever the crossfade logic expects later —
no crossfade is needed here since nothing is on screen yet to preserve.
**How it behaves now:** unchanged visually; still the first layer added on map load.

### 3. Opacity effect reads the visible slot — `web/src/components/map/useMapContextLayers.ts`

**Before:**
```ts
if (map.getLayer("flood-raster")) map.setPaintProperty("flood-raster", "raster-opacity", floodOpacity);
```

**After:**
```ts
const id = visibleFloodLayerIdRef.current;
if (map.getLayer(id)) map.setPaintProperty(id, "raster-opacity", floodOpacity);
```

**What changed:** hardcoded id replaced by the ref read.
**Why:** once the visible layer's id can be either `"flood-raster-0"` or `"flood-raster-1"`, any code
that used the old fixed string would silently do nothing after the first crossfade swap (the layer
`"flood-raster"` would no longer exist).
**How it behaves now:** the opacity slider keeps working correctly no matter how many step changes
(and therefore slot swaps) have already happened.

### 4. DEM `beforeId` stacking anchor reads the visible slot — `web/src/components/map/useMapContextLayers.ts`

**Before:**
```ts
map.addLayer(
  { id: "dem-raster", type: "raster", source: "dem-raster", paint: { "raster-opacity": demOpacityRef.current } },
  map.getLayer("flood-raster") ? "flood-raster" : (map.getLayer("rivers-line") ? "rivers-line" : undefined),
);
```

**After:**
```ts
const floodId = visibleFloodLayerIdRef.current;
map.addLayer(
  { id: "dem-raster", type: "raster", source: "dem-raster", paint: { "raster-opacity": demOpacityRef.current } },
  map.getLayer(floodId) ? floodId : (map.getLayer("rivers-line") ? "rivers-line" : undefined),
);
```

**What changed:** `"flood-raster"` literal replaced by `floodId` (the current visible slot).
**Why:** DEM must always render *below* the flood raster (existing stacking rule from 2026-08-04); if
this still checked the old fixed id, `map.getLayer("flood-raster")` would return `undefined` after
the first crossfade swap, and the DEM layer would silently fall back to stacking under `rivers-line`
instead — putting DEM *above* the currently-visible flood slot and reintroducing the exact bug this
anchor was added to prevent.
**How it behaves now:** correct stacking is preserved across every step change, not just at mount.

### 5. Step-change effect rewritten as a 2-slot crossfade — `web/src/components/map/useMapContextLayers.ts`

**Before:**
```ts
const apply = () => {
  if (map.getLayer("flood-raster")) map.removeLayer("flood-raster");
  if (map.getSource("flood-raster")) map.removeSource("flood-raster");
  if (!floodCogKey) return;
  const tileUrl = buildCogTileUrl(floodTileServerBase(), floodCogKey, floodColormap);
  if (!tileUrl) return;
  map.addSource("flood-raster", { type: "raster", tiles: [tileUrl], tileSize: 256 });
  map.addLayer({
    id: "flood-raster", type: "raster", source: "flood-raster",
    paint: { "raster-opacity": floodOpacity },
  }, map.getLayer("rivers-line") ? "rivers-line" : undefined);
};
```

**After** (key part; full logic also handles the "no valid step" and "user drags timeline faster than
tiles can load" cases — see file):
```ts
const targetId = pendingFloodLayerIdRef.current ?? (visibleId === "flood-raster-0" ? "flood-raster-1" : "flood-raster-0");
if (map.getLayer(targetId)) map.removeLayer(targetId);
if (map.getSource(targetId)) map.removeSource(targetId);

map.addSource(targetId, { type: "raster", tiles: [tileUrl], tileSize: 256, maxzoom: 18 });
map.addLayer({
  id: targetId, type: "raster", source: targetId,
  paint: { "raster-opacity": floodOpacity },
}, map.getLayer("rivers-line") ? "rivers-line" : undefined);
pendingFloodLayerIdRef.current = targetId;

const onSourceData = (e: maplibregl.MapSourceDataEvent) => {
  if (e.sourceId !== targetId || !e.isSourceLoaded) return;
  map.off("sourcedata", onSourceData);
  if (pendingFloodLayerIdRef.current !== targetId) return; // slot was reused by a newer step change mid-load
  if (visibleFloodLayerIdRef.current !== targetId) {
    const old = visibleFloodLayerIdRef.current;
    if (map.getLayer(old)) map.removeLayer(old);
    if (map.getSource(old)) map.removeSource(old);
  }
  visibleFloodLayerIdRef.current = targetId;
  pendingFloodLayerIdRef.current = null;
};
map.on("sourcedata", onSourceData);
```

**What changed:** the old code always operated on the single id `"flood-raster"` and removed it
synchronously before adding the replacement. The new code (a) picks the *other* slot (or reuses an
already-loading slot if the user changed steps again before it finished), (b) adds the new
source/layer into that slot while the visible slot is left untouched, (c) only deletes the old
visible slot inside a `sourcedata` listener that fires once the new slot's tiles have actually
finished loading (`e.isSourceLoaded === true`), and (d) guards against a stale listener firing for a
slot that was since reused by an even newer step change.
**Why:** this is the direct fix for the symptom — deleting the currently-displayed layer before its
replacement exists always produces one fully empty frame. The old comment justified remove-then-add
as "cancels the old request" (true — removing a loading source does abort its in-flight tile
requests), but request cancellation and visual continuity are two different requirements, and only
the first one was actually satisfied.
**How it behaves now:** changing the timeline step no longer removes anything from the screen until
the replacement tiles are ready; the old raster stays visible (crossfades to the new one) instead of
flashing empty. Rapid successive step changes (dragging the slider) reuse the same pending slot
rather than accumulating a 3rd/4th layer, so no leak and no runaway request count.

## 5. How to find this again

- `grep -rn "visibleFloodLayerIdRef\|pendingFloodLayerIdRef" web/src/components/map/useMapContextLayers.ts`
- `grep -rn "flood-raster-0\|flood-raster-1" web/src`
- `grep -n "sourcedata\|isSourceLoaded" web/src/components/map/useMapContextLayers.ts`
- Route: `/gis-map`. Plan reference:
  `plans/260817-0900-flood-raster-ingest-system/phase-04-web-db-driven-rendering.md` ("Prefetch —
  BẮT BUỘC" section — the requirement this fix finally satisfies).
- Related, separate report from the same day (do not confuse): `SESSION-2026-08-17-tile-server-perf-investigation.md` — investigates raw tile *request latency*, not the layer-swap stutter.

## 6. Concepts introduced

### MapLibre `raster` source has no atomic "swap URL" API
- **Plain definition:** unlike an `image` source (`updateImage({url, coordinates})`, one call, no
  gap), a tiled `raster` source's tile template can only be changed by removing and re-adding the
  source/layer — MapLibre does not expose a stable `setTiles()` across versions.
- **Why it shows up here:** this is the root architectural reason bake-PNG (image source) felt
  seamless and the titiler/COG migration (raster source) introduced a visible gap — the same "swap
  to a new step" operation requires fundamentally different code for each source type.

### Crossfade via 2 ping-pong layer ids
- **Plain definition:** keep two layer/source slots with fixed alternating ids; always load the new
  content into whichever slot isn't currently visible, and only tear down the old visible slot after
  confirming (via a load event) that the new one is ready.
- **Why it shows up here:** it's the standard workaround for "no atomic swap API" — used any time a
  tile-based layer needs to change source without a visible flash (also common in MapLibre/Mapbox
  GL raster-tile timelapse implementations elsewhere).

### `sourcedata` + `isSourceLoaded` as a load-completion signal
- **Plain definition:** MapLibre fires a `sourcedata` event repeatedly as tiles for a source arrive;
  `e.isSourceLoaded` is `true` only once all currently-required tiles for the viewport have finished
  loading.
- **Why it shows up here:** it's the only reliable "the new layer is actually ready to show" signal
  available for a `raster` source — there's no single `Promise`-returning "add and wait" API.

## 7. Where it got stuck

**False lead — "the colormap encoding step (titiler-side) is the heavy/slow part."**
- Symptom (user's own hypothesis): stutter appears with titiler/COG but not with pre-baked colored
  PNGs, and colormapping is a step the old PNG pipeline didn't need — reasonable on its face.
- Investigation: `curl` timing of the same tile with vs. without the `colormap` query param, 3 runs
  each. Without: 0.130s / 0.051s / 0.053s. With: 0.051s / 0.053s / 0.052s. Effectively identical once
  the first (likely cold) request is discounted.
- Conclusion: **ruled out by direct measurement**, not by reasoning alone. Server-side colormap
  application (a per-pixel lookup-table operation in rio-tiler/GDAL) is cheap relative to everything
  else in the tile request; it is not a candidate worth removing.

**Real cause — confirmed by reading code, not inferred.**
- Symptom: every timeline step change produced one visible "blank flash" before the new raster
  painted, distinct from ordinary tile-loading latency.
- Investigation: read the step-change effect in `useMapContextLayers.ts` directly. It called
  `removeLayer`/`removeSource` on the single id `"flood-raster"` *before* calling `addSource`/
  `addLayer` for the replacement — there is no ambiguity here, this is exactly what the code did, not
  a guess.
- **This is a self-inflicted regression, not a titiler/COG architecture limitation.** The original
  Pha 4 plan already specified the correct sequencing ("giữ layer cũ tới khi tile mới load xong rồi
  mới gỡ") but the implementation done earlier the same day skipped it, apparently trading
  correctness for a simpler one-shot remove-then-add. The user only surfaced this by directly
  comparing against the retired bake-PNG UX, which never had this gap because `image` sources swap
  atomically.
- Fix: 2-slot crossfade described in §4, item 5.

## 8. Verify

```bash
cd web
npx tsc --noEmit -p .
npm run build
```
Expected: both exit 0. `tsc` produced no errors; `npm run build` succeeded with only a pre-existing
chunk-size warning (unrelated to this change, present before this session).

**Not yet verified visually in this session** — no browser was available in this environment. Still
required: open `/gis-map`, drag the timeline slider through several steps (including rapid dragging
faster than tiles can load), and confirm the flood raster crossfades smoothly with no blank flash and
no visible flicker, matching the old bake-PNG feel the user is comparing against. Also confirm DEM
still stacks correctly below the flood raster when both are visible (regression risk from item 4 in
§4).

## 9. Gotchas

- If a third code path ever adds/removes/reads the flood raster layer by a hardcoded id string
  (`"flood-raster"`), it will silently stop working after the first step change, because that literal
  id no longer exists once the ping-pong slots take over — always read `visibleFloodLayerIdRef.current`
  instead.
- The `sourcedata` listener is attached per step-change call and removes itself (`map.off`) once it
  fires for its own `targetId` — but if `apply()` is ever changed to return early *before* reaching
  `map.on("sourcedata", onSourceData)` (e.g. a future guard clause), the corresponding slot would be
  left in `pendingFloodLayerIdRef` forever with no listener to resolve it, silently breaking future
  step changes' slot-reuse logic. Any new early return added to this effect must be placed before
  `addSource`/`addLayer`, not after.
- The stale-listener guard (`if (pendingFloodLayerIdRef.current !== targetId) return;`) is what makes
  rapid slider dragging safe — removing it would let an old, already-superseded `sourcedata` event
  overwrite `visibleFloodLayerIdRef` with an outdated slot after a newer one has already taken over.
- This fix does not touch the tile-request-latency question investigated in the other same-day
  report (`SESSION-2026-08-17-tile-server-perf-investigation.md`) — a slow network/backend can still
  make the *new* tiles take a while to finish loading; crossfading only removes the *empty-frame* gap,
  it does not make individual tile requests faster.
