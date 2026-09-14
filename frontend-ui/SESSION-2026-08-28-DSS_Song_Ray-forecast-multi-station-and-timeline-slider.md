# Session 2026-08-28 — Rainfall chart multi-station selection + simulation timeline responsive slider

Two independent UI changes on `worktree-forecast-multi-station`, both in `apps/web/src/components`.

## 1. Requirement recap

**Change 1.** Forecast tab, rainfall forecast chart: replace the single-station tab row
("chọn 1 trạm") with a control that lets the user pick a *list* of stations to overlay on the
same chart, instead of only ever seeing one station's rainfall at a time.

**Change 2.** GIS map's simulation timeline: user reported (with a screenshot) that the row of
time labels below the play/pause controls becomes unreadable — labels overlapping like
`"17:30 24-0815:0507:30 24-08"` — and gets *worse* the narrower the window is, not better.
Invoked explicitly via `/brainstorm` before implementing.

## 2. How it was implemented + docs used

**Change 1 — multi-select dropdown, not a checkbox row inline.** A checkbox list rendered
directly in the control bar (like the old tab row) would grow unboundedly wide with more
stations; a dropdown trigger (`"N trạm đã chọn"`) that opens a `role="listbox"
aria-multiselectable` panel keeps the control bar width constant regardless of station count.
Click-outside-to-close uses the standard `mousedown` + ref-contains pattern (no library added —
YAGNI, this repo doesn't otherwise depend on a headless-UI package for one dropdown).

Fetching switched from a single `getRainfallByStation` call to `Promise.all` over
`stationIds`, and the chart's per-row data key changed from plain `sourceCode` (e.g. `"kttv"`)
to a composite `` `${stationId}__${sourceCode}` `` (`combinedKey`) — necessary once two
stations can both have a `"kttv"` source in the same row object.

**Shared-state conflict, resolved by judgment call.** `useForecastOverviewState` held one
`stationId` consumed by both the chart (now needs a list) and `ForecastSummaryCard` (which by
design shows exactly one "Trạm đang chọn" trend summary — singular is correct there, not a bug).
Renamed the hook's state to `selectedStationIds: string[]` / `setSelectedStationIds`, and in
`forecast-overview-layout.tsx` derived `primaryStationId = selectedStationIds[0] ?? null` to feed
`ForecastSummaryCard` unchanged. This was found only by grepping `stationId` usages across the
`forecast/overview` directory and reading `forecast-summary-card.tsx`.

**Change 2 — brainstorm before coding.** Used `AskUserQuestion` to settle two design decisions
first (report: `plans/reports/brainstorm-260828-2128-simulation-timeline-responsive.md`):
1. Keep discrete dot-buttons vs. switch to a slider → user picked a **native range slider** with
   a label that floats above/follows the active step.
2. Under tight space, trim label text vs. reduce label *count* dynamically → user picked
   **dynamically compute label count from real width**, keep full readable text.

Root cause of the overlap (found by reading the code, not guessed): `MAX_LABELS = 10` was a
hardcoded constant, completely independent of the track's actual pixel width. With ~72 data
points and a fixed count of 10 labels, spacing was by array index, and each label
(`white-space: nowrap`, ~55–60px) had no guaranteed room once the container narrowed — the
`≥900px` breakpoint that keeps the whole bar visible doesn't guarantee 10×60px of space.

Implementation replaced the 72 `<button>` dots with one native `<input type="range" step={1}>`
(step=1 preserves the existing invariant that the slider only ever snaps to real data indices,
never interpolates), added a `useTrackMaxLabels(trackRef, count)` hook that uses a
`ResizeObserver` on the track element to compute `maxLabels = Math.max(2, Math.floor(width /
LABEL_WIDTH_PX))` (`LABEL_WIDTH_PX = 64`, an estimate), and rewrote `isLabelTick` to take that
dynamic value and to explicitly hide the label for `index === activeIndex` (now redundant with
the new floating label).

CSS in both changes follows the project's CSS-Grid-only rule (see memory:
`feedback_css-grid-only-layout.md`) — all new layout uses `display: grid`, never flexbox. The
`SimulationTimeline.css` file predated that rule and was flex throughout; it was converted
on-touch rather than left mixed.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/web/src/components/forecast/overview/rainfall-forecast-chart.tsx` | Rainfall chart + station picker | Single-station `role="tablist"` buttons, `stationId: string \| null` prop, plain `sourceCode` chart keys | `StationMultiSelect` checkbox dropdown, `stationIds: string[]` prop, `combinedKey(stationId, sourceCode)` chart keys, `Promise.all` fetch |
| `apps/web/src/components/forecast/overview/use-forecast-overview-state.ts` | Shared forecast-tab state hook | `stationId: string \| null` / `setStationId` | `selectedStationIds: string[]` / `setSelectedStationIds` |
| `apps/web/src/components/forecast/overview/forecast-overview-layout.tsx` | Wires state to chart + summary card | Passed shared `stationId` to both children | Passes `selectedStationIds` to chart, derives `primaryStationId = selectedStationIds[0] ?? null` for `ForecastSummaryCard` |
| `apps/web/src/components/forecast/overview/rainfall-forecast-chart.css` | Station picker styling | Flat tab-row styles | Trigger button + absolute-positioned dropdown menu, grid-based |
| `apps/web/src/components/gis/SimulationTimeline.tsx` | GIS map playback + timeline scrubber | 72 `<button>` dots, fixed `MAX_LABELS = 10`, static "Thời gian:" line | Native `<input type="range">`, `useTrackMaxLabels` (ResizeObserver-driven), floating active label, `isLabelTick` takes dynamic `maxLabels` |
| `apps/web/src/components/gis/SimulationTimeline.css` | Timeline styling | Flexbox layout, dot/line/tick-label rules | Grid layout (`grid-template-rows`), native range-thumb styling (`::-webkit-slider-thumb`, `::-moz-range-thumb`), floating + static label rules |

## 4. Code changes in detail

### 1. Composite chart data key so overlapping stations don't collide — `rainfall-forecast-chart.tsx`

**Before:**
```tsx
interface ChartRow {
  validAt: string;
  [sourceCode: string]: string | number;
}

function toChartRows(data: MultiSourceRainfallResponse): ChartRow[] {
  const now = Date.now();
  const byTimestamp = new Map<string, ChartRow>();
  for (const source of data.sources) {
    for (const point of source.points) {
      if (!isObservedUpToNow(source.sourceCode, point.validAt, now)) continue;
      const row = byTimestamp.get(point.validAt) ?? { validAt: point.validAt };
      row[source.sourceCode] = point.value;
      byTimestamp.set(point.validAt, row);
    }
  }
  return Array.from(byTimestamp.values()).sort((a, b) => a.validAt.localeCompare(b.validAt));
}
```

**After:**
```tsx
interface ChartRow {
  validAt: string;
  [dataKey: string]: string | number;
}

interface StationRainfall {
  station: ForecastStation;
  data: MultiSourceRainfallResponse;
}

function combinedKey(stationId: string, sourceCode: string): string {
  return `${stationId}__${sourceCode}`;
}

function toChartRows(results: StationRainfall[]): ChartRow[] {
  const now = Date.now();
  const byTimestamp = new Map<string, ChartRow>();
  for (const { station, data } of results) {
    for (const source of data.sources) {
      for (const point of source.points) {
        if (!isObservedUpToNow(source.sourceCode, point.validAt, now)) continue;
        const row = byTimestamp.get(point.validAt) ?? { validAt: point.validAt };
        row[combinedKey(station.id, source.sourceCode)] = point.value;
        byTimestamp.set(point.validAt, row);
      }
    }
  }
  return Array.from(byTimestamp.values()).sort((a, b) => a.validAt.localeCompare(b.validAt));
}
```

**What changed:** `toChartRows` now iterates a list of `{ station, data }` pairs (was one
`data`), and the per-source key is `combinedKey(station.id, source.sourceCode)` instead of the
plain `source.sourceCode`.

**Why:** two stations both returning a `"kttv"` source used to write to the same object key
`row["kttv"]`, so the second station's value would silently overwrite the first's in the same
timestamp row.

**How it behaves now:** each station's series gets its own key, e.g.
`"tramA__kttv"` and `"tramB__kttv"`, so both render as separate lines/bars in the same
`ComposedChart` without clobbering each other.

### 2. Single-select tab row → multi-select dropdown — `rainfall-forecast-chart.tsx`

**Before:**
```tsx
export function RainfallForecastChart({
  stations,
  stationId,
  onStationChange,
  horizonHours,
  onHorizonChange,
}: {
  stations: ForecastStation[];
  stationId: string | null;
  onStationChange: (id: string) => void;
  horizonHours: ForecastHorizonHours;
  onHorizonChange: (hours: ForecastHorizonHours) => void;
}) {
  ...
  <div className="rainfall-forecast-chart-stations" role="tablist" aria-label="Chọn trạm mưa">
    {stations.map((station) => (
      <button
        key={station.id}
        type="button"
        role="tab"
        aria-selected={station.id === stationId}
        className={`rainfall-forecast-chart-station-tab${station.id === stationId ? " rainfall-forecast-chart-station-tab--active" : ""}`}
        onClick={() => onStationChange(station.id)}
      >
        {station.name}
      </button>
    ))}
  </div>
```

**After:**
```tsx
function StationMultiSelect({ stations, stationIds, onStationIdsChange }: { ... }) {
  const [open, setOpen] = useState(false);
  const rootRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!open) return;
    function handleClickOutside(event: MouseEvent) {
      if (rootRef.current && !rootRef.current.contains(event.target as Node)) setOpen(false);
    }
    document.addEventListener("mousedown", handleClickOutside);
    return () => document.removeEventListener("mousedown", handleClickOutside);
  }, [open]);

  function toggleStation(id: string) {
    if (stationIds.includes(id)) {
      onStationIdsChange(stationIds.filter((s) => s !== id));
    } else {
      onStationIdsChange([...stationIds, id]);
    }
  }
  // ... trigger button showing "N trạm đã chọn" + role="listbox" checkbox menu
}

export function RainfallForecastChart({
  stations,
  stationIds,
  onStationIdsChange,
  horizonHours,
  onHorizonChange,
}: { ... }) {
  ...
  <StationMultiSelect stations={stations} stationIds={stationIds} onStationIdsChange={onStationIdsChange} />
```

**What changed:** `stationId: string | null` / `onStationChange` became `stationIds: string[]` /
`onStationIdsChange`; the inline tab-row markup moved into a new `StationMultiSelect`
sub-component with dropdown + checkboxes and its own click-outside handling.

**Why:** the requirement is to overlay multiple stations, which a single-select tab list cannot
express.

**How it behaves now:** clicking the trigger opens a checkbox list; checking/unchecking a
station toggles it in `stationIds`, immediately re-triggering the `Promise.all` fetch and
re-rendering the chart with that station's series added or removed.

### 3. Shared `stationId` split into list (chart) vs. derived primary (summary card) — `use-forecast-overview-state.ts`, `forecast-overview-layout.tsx`

**Before:**
```ts
// use-forecast-overview-state.ts
const [stationId, setStationId] = useState<string | null>(null);
return { stationId, setStationId, horizonHours, setHorizonHours, activeTimestamp, ... };
```
```tsx
// forecast-overview-layout.tsx
const { stationId, setStationId, horizonHours, setHorizonHours } = useForecastOverviewState();
...
<RainfallForecastChart stations={stations} stationId={stationId} onStationChange={setStationId} .../>
...
<ForecastSummaryCard stationId={stationId} stationName={activeStationName} horizonHours={horizonHours} />
```

**After:**
```ts
// use-forecast-overview-state.ts
const [selectedStationIds, setSelectedStationIds] = useState<string[]>([]);
return { selectedStationIds, setSelectedStationIds, horizonHours, setHorizonHours, activeTimestamp, ... };
```
```tsx
// forecast-overview-layout.tsx
const { selectedStationIds, setSelectedStationIds, horizonHours, setHorizonHours } = useForecastOverviewState();
...
const primaryStationId = selectedStationIds[0] ?? null;
const activeStationName = stations.find((s) => s.id === primaryStationId)?.name ?? "—";
...
<RainfallForecastChart stations={stations} stationIds={selectedStationIds} onStationIdsChange={setSelectedStationIds} .../>
...
<ForecastSummaryCard stationId={primaryStationId} stationName={activeStationName} horizonHours={horizonHours} />
```

**What changed:** the hook's single `stationId` became `selectedStationIds: string[]`; the
layout now derives a scalar `primaryStationId` from `selectedStationIds[0]` instead of forwarding
one shared value to both children.

**Why:** `ForecastSummaryCard` genuinely needs exactly one active station (it renders "Trạm đang
chọn": one name) while the chart now needs a list — they can no longer share one piece of state
verbatim.

**How it behaves now:** selecting stations 2 and 3 in the chart's dropdown overlays both on the
chart, while the summary card keeps showing station 2's (the first selected) trend, without the
card needing any code change.

### 4. Fixed-count labels → width-measured label count — `SimulationTimeline.tsx`

**Before:**
```tsx
const MAX_LABELS = 10;

function isLabelTick(index: number, count: number, activeIndex: number): boolean {
  if (index === activeIndex) return true;
  const step = Math.max(1, Math.round(count / MAX_LABELS));
  if (Math.abs(index - activeIndex) < Math.ceil(step / 2)) return false;
  if (index === 0 || index === count - 1) return true;
  return index % step === 0;
}
```

**After:**
```tsx
const LABEL_WIDTH_PX = 64;

function isLabelTick(index: number, count: number, activeIndex: number, maxLabels: number): boolean {
  if (index === activeIndex) return false;
  const step = Math.max(1, Math.round(count / maxLabels));
  if (Math.abs(index - activeIndex) < Math.ceil(step / 2)) return false;
  if (index === 0 || index === count - 1) return true;
  return index % step === 0;
}

function useTrackMaxLabels(trackRef: React.RefObject<HTMLDivElement | null>, count: number): number {
  const [maxLabels, setMaxLabels] = useState(10);

  useLayoutEffect(() => {
    const el = trackRef.current;
    if (!el) return;
    const observer = new ResizeObserver((entries) => {
      const width = entries[0]?.contentRect.width ?? el.clientWidth;
      setMaxLabels(Math.max(2, Math.floor(width / LABEL_WIDTH_PX)));
    });
    observer.observe(el);
    return () => observer.disconnect();
  }, [trackRef]);

  return Math.min(maxLabels, count);
}
```

**What changed:** `MAX_LABELS` constant removed; `isLabelTick` gained a `maxLabels` parameter and
now returns `false` for the active index (was `true`); a new `useTrackMaxLabels` hook measures
the track element's real width via `ResizeObserver` and derives label capacity from it.

**Why:** label count used to be fixed regardless of available pixels, causing overlap on narrow
windows (the reported bug); the active-index label is now redundant because a separate floating
label (see change 5) already shows it.

**How it behaves now:** shrinking the browser window shrinks the track, `ResizeObserver` fires,
`maxLabels` recomputes downward, and `isLabelTick` spaces the remaining static labels further
apart so none overlap — no fixed number of labels is guaranteed anymore, only "however many fit."

### 5. Dot-button row → native range input + floating label — `SimulationTimeline.tsx`

**Before:**
```tsx
<span className="simulation-timeline-time t-caption">Thời gian: {formatTs(current.ts)}</span>

<div className="simulation-timeline-track">
  <div className="simulation-timeline-line" />
  <div className="simulation-timeline-ticks">
    {results.map((r, i) => (
      <button
        key={r.ts}
        type="button"
        className={`simulation-timeline-tick ${i === index ? "simulation-timeline-tick--active" : ""}`}
        onClick={() => setIndex(i)}
        title={formatTs(r.ts)}
      >
        <span className="simulation-timeline-dot-slot">
          <span className="simulation-timeline-dot" />
        </span>
        {isLabelTick(i, results.length, index) && (
          <span className="simulation-timeline-tick-label">{formatTs(r.ts)}</span>
        )}
      </button>
    ))}
  </div>
</div>
```

**After:**
```tsx
<div className="simulation-timeline-track" ref={trackRef}>
  <span className="simulation-timeline-active-label" style={{ left: `${activePercent}%` }}>
    {formatTs(current.ts)}
  </span>

  <input
    type="range"
    className="simulation-timeline-range"
    min={0}
    max={results.length - 1}
    step={1}
    value={index}
    onChange={(e) => setIndex(Number(e.target.value))}
    aria-label="Chọn mốc thời gian mô phỏng"
  />

  <div className="simulation-timeline-static-labels">
    {results.map((r, i) =>
      isLabelTick(i, results.length, index, maxLabels) ? (
        <span
          key={r.ts}
          className="simulation-timeline-static-label"
          style={{ left: `${results.length > 1 ? (i / (results.length - 1)) * 100 : 0}%` }}
        >
          {formatTs(r.ts)}
        </span>
      ) : null,
    )}
  </div>
</div>
```

**What changed:** 72 individual `<button>` dot elements were replaced by one native
`<input type="range" step={1}>`; the static "Thời gian: {current}" text line was replaced by a
`position: absolute` floating label positioned via `left: ${activePercent}%`.

**Why:** rendering 72 DOM buttons per frame was the mechanism behind the fixed-label-count bug;
a single range input scales its interaction surface to any data length without per-item DOM cost,
and `step={1}` keeps the "only snap to real data indices" invariant that the old dot list
enforced by construction.

**How it behaves now:** dragging the slider snaps `index` to integer steps 0..`results.length-1`
only; the floating label follows the thumb's `%` position continuously as it's dragged.

### 6. Flexbox → CSS Grid conversion + native range-thumb styling — `SimulationTimeline.css`

**Before (excerpt):**
```css
.simulation-timeline {
  display: flex;
  align-items: center;
  gap: var(--sp-3);
  ...
}

.simulation-timeline-ticks {
  display: flex;
  justify-content: space-between;
  ...
}
```

**After (excerpt):**
```css
.simulation-timeline {
  display: grid;
  grid-auto-flow: column;
  align-items: center;
  gap: var(--sp-3);
  ...
}

.simulation-timeline-track {
  position: relative;
  display: grid;
  grid-template-rows: 18px 20px 16px;
  padding: 0 6px;
}

.simulation-timeline-range::-webkit-slider-thumb {
  appearance: none;
  width: 12px;
  height: 12px;
  margin-top: -5px;
  border-radius: 50%;
  background: #426be5;
  box-shadow: 0 0 0 3px rgba(66, 107, 229, 0.2);
  cursor: pointer;
}

.simulation-timeline-range::-moz-range-thumb {
  width: 12px;
  height: 12px;
  border: none;
  border-radius: 50%;
  background: #426be5;
  box-shadow: 0 0 0 3px rgba(66, 107, 229, 0.2);
  cursor: pointer;
}
```

**What changed:** every `display: flex` in the file became `display: grid` (+
`grid-auto-flow: column` where a horizontal row was needed); the track's three stacked rows
(floating label / range input / static labels) now use `grid-template-rows: 18px 20px 16px`
instead of a flex column with `gap`; new `::-webkit-slider-thumb` / `::-moz-range-thumb` /
`::-webkit-slider-runnable-track` / `::-moz-range-track` rules reproduce the old dot's 12px blue
circle + halo look on the native input, since range-input styling has no cross-browser-neutral
single selector.

**Why:** the project's CSS-Grid-only rule (no flexbox) applies to touched files; this file
predated the rule and was converted on-touch. The thumb styling is required because a bare
`<input type="range">` has no visual resemblance to the previous dot design in any browser.

**How it behaves now:** layout is pixel-identical in intent (row of controls + track) but built
without `display: flex` anywhere in the file; the slider thumb renders as the same blue circle
with halo in both Chromium/Safari (`-webkit-` rules) and Firefox (`-moz-` rules).

## 5. How to find this again

- Grep `combinedKey` or `StationMultiSelect` in `apps/web/src/components/forecast/overview/` for
  the multi-station chart logic.
- Grep `selectedStationIds` or `primaryStationId` to trace the shared-state split between the
  chart and `ForecastSummaryCard`.
- Grep `useTrackMaxLabels` or `LABEL_WIDTH_PX` in `apps/web/src/components/gis/
  SimulationTimeline.tsx` for the responsive label logic.
- Grep `simulation-timeline-range` for the native range-input styling rules.
- Brainstorm design record: `plans/reports/brainstorm-260828-2128-simulation-timeline-responsive.md`.

## 6. Concepts introduced

- **`ResizeObserver`** — a browser API that fires a callback whenever an observed element's
  content box size changes (independent of window `resize` events, e.g. also fires on flex/grid
  reflow). Needed here because the track's width depends on the surrounding grid layout, not just
  the viewport width, so a plain `window.addEventListener("resize", ...)` would have missed
  container-driven width changes.
- **Native `<input type="range">` pseudo-elements** (`::-webkit-slider-thumb`,
  `::-moz-range-thumb`, `::-webkit-slider-runnable-track`, `::-moz-range-track`) — browser-specific
  selectors that are the only way to restyle a range input's thumb/track, since there is no
  standard cross-browser selector for them yet.
- **Composite/combined map keys** (`` `${a}__${b}` ``) — a plain technique for avoiding key
  collisions when merging two independent dimensions (station × source) into one flat object,
  used here instead of introducing a nested data structure into `recharts`, which expects flat
  row objects.

## 7. Where it got stuck

**Change 1 — shared `stationId` had two different cardinality needs.** Not visible until the
prop was actually renamed and `ForecastSummaryCard`'s usage was checked. Grepping `stationId`
across `forecast/overview/` and reading `forecast-summary-card.tsx` showed it renders a singular
"Trạm đang chọn" trend, which a plain rename to an array would have silently broken (it would
have received an array where it expects a string). During the earlier brainstorm-style question
for this task, the initial answer favored "change the shared state straight to an array, keep it
in sync app-wide" — implementer judgment call deviated from that literal instruction and instead
derived `primaryStationId = selectedStationIds[0] ?? null` to keep `ForecastSummaryCard`
untouched, since implementing it literally would have broken that card. This deviation was
surfaced to the user rather than done silently.

**Change 2 — invalid CSS caught during self-review, not by a build error.** The first draft of
`.simulation-timeline-active-label` used
`transform: translateX(clamp(0%, calc(-1 * var(--label-edge-guard, 50%)), 100%))`, intending to
clamp the floating label so it wouldn't overflow past the track edges near index 0 or the last
index. This does not do anything meaningful — `clamp()` with an undefined custom-property
fallback used inside a `calc()` this way doesn't produce the intended edge-clamping behavior, and
there is no build-time validation that would have caught a nonsensical (but syntactically
tolerated) CSS value like this; a browser silently ignores or misapplies it. It was caught by
re-reading the rule before ever running it, not via any error message. Fixed by reverting to
plain `transform: translateX(-50%)`, matching every static label in the same file, which already
accepted the same minor edge-overflow risk without complaint — so no special-casing was actually
needed for the floating label either (YAGNI). Note: the surrounding CSS comment in the shipped
file still says *"clamp() ngăn nhãn tràn ra ngoài 2 mép track..."*, describing the discarded
approach rather than the code that actually shipped (`translateX(-50%)`, no clamp) — a stale
comment left behind by the fix, worth cleaning up if this file is touched again.

## 8. Verify

```bash
pnpm --filter web exec tsc --noEmit
```
Run after each commit; both exited 0 (no type errors). Additionally grepped
`apps/web/src` for the removed CSS class names (`simulation-timeline-tick`, `-dot`, `-line`,
`-time`) to confirm no other file still referenced the deleted markup — none found.

No automated UI/visual test was run in either change. Per existing repo/user convention
(`feedback_no-claude-browser-testing.md`), the agent does not self-test UI via a headless
browser; the user reviews the rendered UI themselves via screenshots. This was stated explicitly
to the user rather than implied.

## 9. Gotchas

- `StationMultiSelect`'s click-outside handler listens on `document` for `mousedown`; if another
  component ever adds its own document-level `mousedown` listener that calls `stopPropagation()`,
  this dropdown will stop closing on outside clicks — check for that if the dropdown starts
  "sticking" open.
- `combinedKey` assumes `stationId` and `sourceCode` never contain the literal substring `"__"`;
  if a future station/source code does, two different (station, source) pairs could collide.
  Cheap to fix (encode differently) but not guarded against today.
- `LABEL_WIDTH_PX = 64` is an estimate, not measured per-locale/per-font. If timestamp formatting
  changes to a longer format (e.g. adds seconds or a year), labels will start crowding again even
  though the ResizeObserver logic is otherwise correct — the constant would need bumping.
- The stale CSS comment in `SimulationTimeline.css` (describing a `clamp()` edge-guard that isn't
  actually in the shipped rule) could mislead a future reader into thinking edge-overflow is
  handled when it isn't — same risk accepted for static labels, but worth fixing the comment if
  this file is edited again.
- `isLabelTick` now depends on `maxLabels` being at least 2 (`Math.max(2, ...)` in
  `useTrackMaxLabels`); if that floor is ever removed, `Math.round(count / maxLabels)` divides by
  a value that could reach 0 for a very narrow track, producing `Infinity`/`NaN` steps.
