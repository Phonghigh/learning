# Session 2026-09-04 — Rain station & survey station dot markers

## 1. Requirement recap

User asked (in Vietnamese) to change the map markers with class
`maplibregl-marker maplibregl-marker-anchor-center` to:
1. Round dots colored green.
2. Show rain station names on the map.

After clarifying questions, the user refined this to:
1. Use **blue**, not green.
2. Keep the label idea as planned.
3. Apply the dot+label style to **all** station markers — both regular "stations" and
   "rainStations" — not just rain stations. No marker should keep the old pin style.
4. Exact styling details (size, border, label look) left to my judgment.
5. Explicitly asked me to first check whether other maps/tabs in the app also render markers,
   and build a **reusable function with parameters** rather than duplicating custom marker DOM
   code per map, to avoid future maintenance pain.

## 2. How it was implemented + docs used

Ran a repo-wide grep for `maplibregl.Marker` across `apps/web/src` first, per the user's request
to check other map instances before deciding on a shared-helper approach. Only one file uses it:
`apps/web/src/components/gis/AppMapCanvas.tsx`, with two separate marker-creation loops (one for
`stations.features`, one for `rainStations.features`). No other map component exists in the app
today, so no cross-file refactor was needed — but a shared helper was still built as requested,
so any future map component can reuse the same dot-marker look with one import instead of
re-implementing custom DOM markers.

Options considered:
- **Use MapLibre's built-in `new maplibregl.Marker({ color })`** — rejected because the built-in
  marker is a teardrop pin shape (SVG), not a plain circle, and it has no way to attach an
  always-visible text label next to it. A label would require a separate DOM element manually
  positioned, effectively the same amount of work as a custom marker anyway.
- **Custom DOM element per marker, inlined at each call site** — rejected per the user's explicit
  instruction to avoid duplicating this logic across the two loops (and any future map).
- **Winner: a small shared helper `createDotMarker(options)`** in a new file
  `create-dot-marker.ts`, taking `map`, `lngLat`, `color`, `label`, `popupHtml`, and an optional
  `onClick`. Builds one `<div class="gis-dot-marker">` containing a colored circle `<span>` and a
  label `<span>`, wraps it in `new maplibregl.Marker({ element, anchor: "center" })`. Both loops
  in `AppMapCanvas.tsx` now call this instead of `new maplibregl.Marker(...)`.

For color, `MAP_COLORS.hydrology` (`#2563eb`) was reused from the existing token file
(`apps/web/src/lib/tokens.ts`) instead of inventing a new blue constant, since it already matched
"blue" and kept both marker types visually consistent with the rest of the hydrology-themed UI.

No external docs were needed beyond the existing MapLibre GL JS `Marker` API already in use
elsewhere in the file (`setLngLat`, `setPopup`, `addTo`, `{ element, anchor }` constructor option).

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/web/src/components/gis/create-dot-marker.ts` | New shared helper | did not exist | exports `createDotMarker()` building a colored-dot + label DOM marker |
| `apps/web/src/components/gis/AppMapCanvas.tsx` | Map canvas component, station rendering | each loop built its own `new maplibregl.Marker(...)` (default pin, or `{ color: MAP_COLORS.watershed }` pin for rain stations) | both loops call `createDotMarker({ map, lngLat, color: MAP_COLORS.hydrology, label, popupHtml, onClick? })` |
| `apps/web/src/components/gis/AppMapCanvas.css` | Map canvas styles | only had `.gis-map-canvas` rule | added `.gis-dot-marker`, `.gis-dot-marker__dot`, `.gis-dot-marker__label` rules |

## 4. Code changes in detail

### 1. New shared marker-builder — `apps/web/src/components/gis/create-dot-marker.ts`

**Before:** (file did not exist)

**After:**
```ts
import * as maplibregl from "maplibre-gl";

export interface DotMarkerOptions {
  map: maplibregl.Map;
  lngLat: [number, number];
  color: string;
  label: string;
  popupHtml: string;
  onClick?: () => void;
}

export function createDotMarker({ map, lngLat, color, label, popupHtml, onClick }: DotMarkerOptions): maplibregl.Marker {
  const el = document.createElement("div");
  el.className = "gis-dot-marker";

  const dot = document.createElement("span");
  dot.className = "gis-dot-marker__dot";
  dot.style.backgroundColor = color;

  const text = document.createElement("span");
  text.className = "gis-dot-marker__label";
  text.textContent = label;

  el.appendChild(dot);
  el.appendChild(text);

  const marker = new maplibregl.Marker({ element: el, anchor: "center" })
    .setLngLat(lngLat)
    .setPopup(new maplibregl.Popup().setHTML(popupHtml))
    .addTo(map);

  if (onClick) {
    el.addEventListener("click", onClick);
  }

  return marker;
}
```

**What changed:** New file exporting one function. Builds a plain `<div>` with a circle `<span>`
(background color set inline per call, since color varies per marker type) and a label `<span>`,
instead of relying on MapLibre's built-in pin SVG.

**Why:** MapLibre's default `Marker` renders a teardrop pin and has no label slot. The user
wanted a round dot plus a visible station name, and explicitly asked for one reusable function
instead of copy-pasting this DOM-building logic into both marker loops (and any future map).

**How it behaves now:** Any caller passing `map`, `lngLat`, `color`, `label`, `popupHtml` gets a
marker that looks like a colored dot with a name tag next to it, with the same popup-on-click
behavior as before, plus an optional custom `onClick` for extra logic (used by the station loop
to still fire `onFeatureSelect`).

### 2. Both marker loops switched to the shared helper — `apps/web/src/components/gis/AppMapCanvas.tsx`

**Before (survey station loop, ~line 258-268):**
```tsx
const marker = new maplibregl.Marker()
  .setLngLat([lng, lat])
  .setPopup(new maplibregl.Popup().setHTML(`<strong>${name}</strong><br/>${stationType}`))
  .addTo(map!);
marker.getElement().addEventListener("click", () => {
  onFeatureSelectRef.current?.({ name, stationType, code });
});
stationMarkersRef.current.push(marker);
```

**After:**
```tsx
const marker = createDotMarker({
  map: map!,
  lngLat: [lng, lat],
  color: MAP_COLORS.hydrology,
  label: name,
  popupHtml: `<strong>${name}</strong><br/>${stationType}`,
  onClick: () => onFeatureSelectRef.current?.({ name, stationType, code }),
});
stationMarkersRef.current.push(marker);
```

**Before (rain station loop, ~line 273-281):**
```tsx
const marker = new maplibregl.Marker({ color: MAP_COLORS.watershed })
  .setLngLat([lng, lat])
  .setPopup(new maplibregl.Popup().setHTML(`<strong>${name}</strong>`))
  .addTo(map!);
```

**After:**
```tsx
const marker = createDotMarker({
  map: map!,
  lngLat: [lng, lat],
  color: MAP_COLORS.hydrology,
  label: name,
  popupHtml: `<strong>${name}</strong>`,
});
rainStationMarkersRef.current.push(marker);
```

Also added `import { createDotMarker } from "./create-dot-marker";` near the top of the file.

**What changed:** Both `new maplibregl.Marker(...)` pin constructions replaced by
`createDotMarker(...)` calls. The station loop's separate `.getElement().addEventListener("click", ...)`
call was folded into the `onClick` option instead of being attached after construction. Both loops
now use the same `MAP_COLORS.hydrology` blue (previously the rain-station loop used
`MAP_COLORS.watershed`, a different color, and the station loop used no color option at all,
i.e. the default MapLibre pin red/blue).

**Why:** The user required every station marker — regular and rain — to use the same dot+label
style and the same blue, with no marker left on the old pin style.

**How it behaves now:** Both survey stations and rain stations render as small blue circles with
their name shown as a label pill next to the dot, clicking still opens the popup (MapLibre default
behavior for markers with `.setPopup()`) and, for survey stations, still triggers
`onFeatureSelect`.

### 3. New marker CSS — `apps/web/src/components/gis/AppMapCanvas.css`

**Before:** only contained the `.gis-map-canvas` rule (map container sizing/border-radius).

**After (appended):**
```css
.gis-dot-marker {
  display: flex;
  align-items: center;
  gap: 4px;
  cursor: pointer;
}

.gis-dot-marker__dot {
  width: 10px;
  height: 10px;
  border-radius: 50%;
  border: 1.5px solid #ffffff;
  box-shadow: 0 0 2px rgba(0, 0, 0, 0.5);
  flex-shrink: 0;
}

.gis-dot-marker__label {
  font-size: 11px;
  font-weight: 600;
  color: #1e293b;
  background: rgba(255, 255, 255, 0.85);
  padding: 1px 4px;
  border-radius: 3px;
  white-space: nowrap;
  pointer-events: none;
}
```

**What changed:** Three new rules added, nothing removed.

**Why:** Styling details were left to my judgment (point 4 of the requirement). A 10px circle
with a white border and subtle shadow keeps the dot visible against any basemap; the label uses
a semi-opaque white background pill so text stays legible over dark map tiles, and
`white-space: nowrap` keeps station names from wrapping awkwardly.

**How it behaves now:** `.gis-dot-marker__label` has `pointer-events: none` so clicks always land
on the marker's containing div/dot rather than being intercepted by the text span — this keeps
the popup-open and `onClick` behavior reliable regardless of where on the label the user clicks.

## 5. How to find this again

- Function: `createDotMarker` in `apps/web/src/components/gis/create-dot-marker.ts`
- CSS classes: `.gis-dot-marker`, `.gis-dot-marker__dot`, `.gis-dot-marker__label` in
  `apps/web/src/components/gis/AppMapCanvas.css`
- Call sites: grep `createDotMarker(` in `apps/web/src/components/gis/AppMapCanvas.tsx`
  (survey station loop and rain station loop)
- Color token: `MAP_COLORS.hydrology` in `apps/web/src/lib/tokens.ts`
- Commit: `ba567a8` — `feat(web): show station markers as labeled blue dots`

## 6. Concepts introduced

- **MapLibre custom marker element**: MapLibre's `Marker` constructor accepts an `element` option
  — any DOM node — instead of using its default pin SVG. This is how the dot+label combo was
  built; the library then just handles positioning/anchoring that element at the given `lngLat`.
  Needed here because the built-in pin marker has no label slot and isn't a circle.
- **`anchor: "center"`**: tells MapLibre to position the custom element so its own center (not its
  top-left corner, which is the default anchor for pin-shaped markers) sits at the geographic
  coordinate — important for a circular dot marker so the dot itself, not some corner of the div,
  lines up with the station's actual location.

## 7. Where it got stuck

No real snags this session. The one risk point — whether other map components elsewhere in the
app also needed the same refactor — was resolved up front by grepping `maplibregl.Marker` across
`apps/web/src`, which returned only `AppMapCanvas.tsx`. That confirmed the shared-helper approach
was correctly scoped without needing a broader multi-file change. The TypeScript build
(`pnpm --filter web run build`) passed on the first attempt with no type errors, so there was no
compile-error chase to document here.

## 8. Verify

```
pnpm --filter web run build
```
Ran inside the worktree (`E:\DSS\DSS_Song_Ray\.claude\worktrees\rain-station-dot-markers`).
Output: `tsc -b && vite build` completed successfully, producing dist output with only a
pre-existing unrelated chunk-size warning (not related to this change). No type errors.

Visual verification was intentionally **not** done via Claude-driven browser automation, per this
project's existing memory rule (no Claude-driven browser testing — rely on `tsc`/build plus the
user's own screenshots at `localhost:5173`). The user should visually confirm dot color/label
placement on both station and rain-station layers.

## 9. Gotchas

- `MAP_COLORS.hydrology` is now used for **both** station types. If a future request asks to
  differentiate survey stations from rain stations visually again, change the `color` argument
  passed to each `createDotMarker(...)` call site — don't add a second CSS class, the color is
  intentionally passed as an inline style so it stays per-instance without needing new CSS rules
  per color.
- `.gis-dot-marker__label` has `pointer-events: none` on purpose — removing it would let clicks on
  the label text miss the marker's click handler in some browsers, breaking `onFeatureSelect` for
  survey stations.
- `createDotMarker` always calls `.setPopup(...)`, so every marker gets a popup even if a caller
  doesn't want one — there is no way to opt out of the popup with the current signature. Future
  callers that want a dot without a popup would need the function extended with an optional flag.
- The helper lives under `components/gis/`, not a generic `lib/` or `utils/` folder — if a
  non-map component ever needs a similar dot marker, this is the first place to look before
  duplicating it.
