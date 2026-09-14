# Session 2026-08-27 — Rebuild `/operation-log` as a reservoir monitoring dashboard

## 1. Requirement recap

Replace the plain event-log table page at `/operation-log` with a desktop-first "Real-time
Reservoir Operations Monitoring and Audit Dashboard": dark navy control-room visual style scoped to
just this page (rest of the app is light-themed), 4 KPI cards, a filter/action toolbar, a two-column
(57/43) workspace with an event timeline + plan-vs-actual chart on the left and a decision summary +
live SCADA metrics grid (4x2) + related documents panel + comparison table on the right, linked
interactions between the metric cards / chart tabs / comparison table / event detail drawer,
accessibility (icon+text pairing, aria-labels, focus-visible, table headers), and responsive
behavior including the 2-col workspace collapsing to 1 col under 1400px. Backend endpoints for
SCADA telemetry, documents, decisions, and chart series don't exist yet, so this data is a local
mock dataset — a deliberate scope call, documented in `mock-data.ts`'s file comment, not an
oversight.

## 2. How it was implemented + docs used

The route (`/operation-log`, behind `RequireAuth` + `AppShell`) already existed in `App.tsx`, so no
routing/navigation changes were needed — only the page content and its component tree.

Ten new presentational components were added under
`apps/web/src/components/operation-log/` (types, mock-data, kpi-row, filter-toolbar,
event-timeline, decision-card, metrics-grid, documents-panel, discharge-chart, comparison-table,
event-detail-drawer), each under ~130 lines, composed inside a rewritten `OperationLogPage.tsx`
that owns the shared interaction state (active chart metric, hovered timestamp, selected event for
the drawer).

Theming decision: rather than touching the shared `styles.css` tokens (which the rest of the app
depends on for its light theme), a full dark palette was defined as `--ops-*` custom properties
scoped under a `.ops-dashboard` root class in `OperationLogPage.css`. This keeps the dark theme
fully local to one page with zero risk of leaking into KpiCard.css or any other shared component.

Chart library: Recharts, reusing the project's existing `CHART_DOT_PROPS` / `CHART_LINE_STROKE_WIDTH`
tokens from `lib/tokens.ts` rather than inventing new ones. Straight (non-smoothed) line segments
were chosen deliberately over a curved interpolation — operational schedules are stepwise, not
organic, so smoothing would visually invent values that were never planned or measured.

Verification used the `agent-browser` CLI for a real rendered screenshot + click pass, since this
was a visual/interaction-heavy page that a type-check alone wouldn't validate.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/web/src/pages/OperationLogPage.tsx` | Page shell | Fetched a paginated event-log list from a service, rendered a sortable/filterable table with loading/error states | Composes 10 dashboard components, owns shared interaction state (active chart metric, hovered timestamp, selected event), reads mock data directly |
| `apps/web/src/pages/OperationLogPage.css` | Page styles | ~80 lines, used the shared light theme | ~700 lines, defines a `.ops-dashboard`-scoped dark palette (`--ops-*` vars), full-bleed grid layout, 1400px collapse breakpoint |
| `apps/web/src/components/operation-log/*.tsx` (10 files, new) | Dashboard building blocks | did not exist | KPI row, filter toolbar, event timeline, decision card, metrics grid, documents panel, discharge chart, comparison table, event detail drawer, shared `types.ts` |
| `apps/web/src/components/operation-log/mock-data.ts` (new) | Local mock dataset | did not exist | SCADA metrics, documents, decision, events, chart series — file comment explicitly notes this is pending real backend endpoints |
| `apps/web/src/App.tsx` | Routing | `/operation-log` already wired behind `RequireAuth` + `AppShell` | unchanged (a temporary unauthenticated `/_preview-operation-log` route was added for local QA and removed before commit — confirmed via `git status` showing this file un-modified in the final diff) |

## 4. Code changes in detail

### 1. Cancelling the shell's page padding for a full-bleed dark theme — `apps/web/src/pages/OperationLogPage.css`

**Before:**
```css
.operation-log-page {
  /* ~80 lines total, light-theme table page */
}
```

**After:**
```css
/* Control-room monitoring theme, scoped to this page only — the rest of the
   app stays on the light theme in styles.css. */
.ops-dashboard {
  --ops-bg: #0b1220;
  --ops-surface: #121b2e;
  --ops-surface-2: #17233a;
  --ops-border: #24314c;
  --ops-border-strong: #34456a;
  --ops-text-1: #e7edf7;
  --ops-text-2: #b7c3d9;
  --ops-text-3: #8494b0;
  --ops-text-4: #5f7093;
  --ops-blue: #3b82f6;
  --ops-green: #22c55e;
  --ops-yellow: #eab308;
  --ops-orange: #f97316;
  --ops-red: #ef4444;

  /* Cancels the light-theme page-root padding so this dashboard owns its
     own full-bleed dark surface and fits inside the shell without an
     extra scrollbar around it. */
  margin: -20px -24px -40px;
  height: calc(100% + 60px);
  display: grid;
  grid-template-rows: auto auto 1fr;
  gap: var(--sp-3);
  padding: var(--sp-3);
  background: var(--ops-bg);
  color: var(--ops-text-1);
  font-size: 13px;
  min-height: 0;
  overflow: hidden;
}

@media (max-width: 1400px) {
  .ops-workspace {
    grid-template-columns: 1fr;
  }
  .ops-dashboard {
    height: auto;
    min-height: 100dvh;
    overflow: visible;
  }
}
```

**What changed:** the page's root selector switched from `.operation-log-page` to `.ops-dashboard`,
which now defines its own `--ops-*` color tokens and hardcodes negative margins / a `calc(100% + 60px)`
height to cancel `AppShell.css`'s `.page-root { padding: var(--pad-page); overflow: auto; }` (where
`--pad-page` is the shorthand `20px 24px 40px`).

**Why:** `AppShell`'s `.page-root` applies padding and its own scroll container to every page, which
is correct for the light-themed pages but leaves an unwanted padding gap and double-scrollbar around
a full-bleed dark dashboard. The dashboard needed to reclaim that space and manage its own internal
scrolling/height.

**How it behaves now:** the dashboard fills the shell edge-to-edge with no visible padding seam, and
below 1400px viewport width the workspace grid collapses to a single column and the page reverts to
natural document flow (`height: auto`, `overflow: visible`) instead of a fixed viewport-relative
height, since the 2-col layout no longer needs to fit everything in one screen.

### 2. `useState` instead of `const` to get proper union-type narrowing — `apps/web/src/pages/OperationLogPage.tsx`

**Before:**
```ts
// (old file — fetched real data, no ConnectionState concept existed)
type FetchState = "loading" | "ready" | "error";
type SortKey = "occurredAt" | "eventType" | "operatorName";
```

**After:**
```ts
import type { ChartMetricKey, ConnectionState, OperationEvent } from "../components/operation-log/types";
...
const [scadaConnection] = useState<ConnectionState>("live");
```

**What changed:** the SCADA connection status is held in `useState<ConnectionState>("live")` (a
proper piece of state, even though nothing currently sets it besides its initializer) rather than a
plain `const scadaConnection: ConnectionState = "live"`.

**Why:** a `const x: "a" | "b" | "c" = "a"` gets narrowed by TypeScript's control-flow analysis to
the literal type `"a"` at the point of declaration. Later code comparing `scadaConnection === "stale"`
then trips TS2367 ("this comparison appears to be unintentional because the types have no overlap"),
because the compiler believes the variable can only ever be `"a"`. Wrapping it in `useState<T>()`
makes the compiler treat the value as the full union type `T`, since `useState`'s return type is
generic over the type parameter and isn't narrowed by the initial argument.

**How it behaves now:** the stale/lost SCADA connection banners' conditional rendering
(`scadaConnection === "stale"` / `=== "lost"`) type-checks cleanly, and the state is structurally
ready for a future real SCADA connection-status source to call `setScadaConnection(...)`.

### 3. Moving the hover handler from `<Tooltip>` to `<LineChart>` — `apps/web/src/components/operation-log/discharge-chart.tsx`

**Before (attempted, not committed — this was fixed before the working commit landed):**
```tsx
<Tooltip
  onMouseMove={(state) => { ... }}
  contentStyle={{ ... }}
/>
```

**After:**
```tsx
<LineChart
  data={data}
  margin={{ top: 8, right: 16, left: -16, bottom: 0 }}
  onMouseLeave={() => onHoverTimestamp(null)}
  onMouseMove={(state) => {
    const label = state?.activeLabel;
    if (typeof label === "string") onHoverTimestamp(label);
  }}
>
  <CartesianGrid stroke="#24314c" strokeDasharray="3 3" vertical={false} />
  <XAxis dataKey="timestamp" fontSize={11} stroke="#8494b0" />
  <YAxis fontSize={11} stroke="#8494b0" domain={["auto", "auto"]} />
  {exceedRanges.map((range, i) => (
    <ReferenceArea key={i} x1={range.from} x2={range.to} fill="#ef4444" fillOpacity={0.12} strokeOpacity={0} />
  ))}
  <Tooltip
    contentStyle={{ background: "#121b2e", border: "1px solid #34456a", borderRadius: 6, fontSize: 12 }}
    labelStyle={{ color: "#e7edf7" }}
  />
  ...
</LineChart>
```

**What changed:** `onMouseMove` / `onMouseLeave` moved from Recharts' `<Tooltip>` element to its
parent `<LineChart>` element; the handler now reads `state?.activeLabel` from the chart's own mouse
event payload instead of a nonexistent Tooltip prop.

**Why:** Recharts' `<Tooltip>` component doesn't accept an `onMouseMove` prop at all — passing one
is a TypeScript error (the component's prop type has no such member). The chart's mouse-move
tracking (needed to feed the "hovering the chart updates the comparison table's timestamp label"
interaction) has to be wired on the chart container itself, which Recharts already exposes an
`activeLabel` field on for exactly this purpose.

**How it behaves now:** hovering anywhere over the chart body calls `onHoverTimestamp(label)` with
the currently-crossed x-axis timestamp (or `null` on mouse-leave), which `OperationLogPage.tsx`
threads into the comparison table to highlight/label the matching row.

### 4. Narrowing `rangeStart` before pushing to a `string`-typed array — `apps/web/src/components/operation-log/discharge-chart.tsx`

**Before (intermediate compile error, fixed in the same session before commit):**
```ts
const exceedRanges: Array<{ from: string; to: string }> = [];
let rangeStart: string | null = null;
data.forEach((point, i) => {
  const exceeds = point.plan != null && point.actual != null && Math.abs(point.actual - point.plan) > tolerance;
  if (exceeds && rangeStart === null) rangeStart = point.timestamp;
  if (!exceeds && rangeStart !== null) {
    exceedRanges.push({ from: rangeStart, to: data[i - 1].timestamp }); // TS2322: string | null not assignable to string
    rangeStart = null;
  }
});
```

**After:**
```ts
const exceedRanges: Array<{ from: string; to: string }> = [];
let rangeStart: string | null = null;
data.forEach((point, i) => {
  const exceeds = point.plan != null && point.actual != null && Math.abs(point.actual - point.plan) > tolerance;
  if (exceeds && rangeStart === null) rangeStart = point.timestamp;
  if (!exceeds && rangeStart !== null) {
    exceedRanges.push({ from: rangeStart, to: data[i - 1].timestamp });
    rangeStart = null;
  }
  if (exceeds && rangeStart !== null && i === data.length - 1) {
    exceedRanges.push({ from: rangeStart, to: point.timestamp });
  }
});
```

**What changed:** every `exceedRanges.push({ from: rangeStart, ... })` call site is now guarded by
an explicit `rangeStart !== null` check in the same `if` condition it's inside.

**Why:** `rangeStart` is declared `string | null`, but `exceedRanges` requires `{ from: string; ... }`.
TypeScript's narrowing only applies within the scope where the null-check was actually performed —
the original code checked `rangeStart !== null` as part of one condition but then referenced
`rangeStart` again later without a fresh check on some paths, so the compiler kept the wider
`string | null` type at the push call.

**How it behaves now:** the "exceeds tolerance" red reference-band ranges on the chart compute
without a type error, including the edge case where a tolerance-exceeding streak runs all the way to
the last data point (closed off with the final point's own timestamp instead of being dropped).

## 5. How to find this again

- Route: `/operation-log` in `apps/web/src/App.tsx`
- Dark theme scope class: `grep -rn "ops-dashboard" apps/web/src/pages/OperationLogPage.css`
- `--ops-*` CSS custom properties: same file, top of `.ops-dashboard` block
- Mock dataset + its scope-call comment: `apps/web/src/components/operation-log/mock-data.ts`
- Chart hover wiring: `grep -n "onMouseMove\|activeLabel" apps/web/src/components/operation-log/discharge-chart.tsx`
- Connection-state banners: `grep -n "ConnectionState\|scadaConnection" apps/web/src/pages/OperationLogPage.tsx`
- 1400px collapse breakpoint: `grep -n "1400px" apps/web/src/pages/OperationLogPage.css`
- Shell padding being cancelled: `apps/web/src/components/layout/AppShell.css`, `--pad-page` token

## 6. Concepts introduced

- **Scoped CSS custom properties**: defining a set of `--ops-*` variables under one root class
  (`.ops-dashboard`) instead of the global `:root`/`styles.css` tokens, so a page-local theme can't
  leak into or be overridden by the app-wide theme. Needed here because the app is light-only
  everywhere else and this one page needed a genuinely separate dark palette.
- **TypeScript literal narrowing on `const`**: a `const` initialized to one member of a union type
  gets narrowed to that literal for later comparisons, which can produce false "no overlap" errors;
  `useState<T>()` (or an explicit type assertion) avoids this because the value is treated as the
  declared generic type, not the literal it happened to start as.
- **Recharts' mouse-event payload (`activeLabel`)**: Recharts chart containers (`<LineChart>`,
  `<AreaChart>`, etc.) expose the currently-hovered x-axis value via `state.activeLabel` in their own
  `onMouseMove` handler — this is the supported way to sync external UI (like a table) to chart
  hover position; `<Tooltip>` itself has no such prop.

## 7. Where it got stuck

- **Symptom:** TS2367 "this comparison appears to be unintentional because the types '\"live\"' and
  '\"stale\"' have no overlap" on `scadaConnection === "stale"`.
  **Cause:** `scadaConnection` was declared as a plain `const scadaConnection: ConnectionState = "live"`,
  which TypeScript narrows to the literal type `"live"` for control-flow purposes.
  **Fix:** switched to `useState<ConnectionState>("live")`, which returns a value typed as the full
  `ConnectionState` union, not the narrowed literal.

- **Symptom:** TS error passing `onMouseMove` to Recharts' `<Tooltip>` component — no such prop on
  its type.
  **Cause:** Recharts' `<Tooltip>` genuinely has no `onMouseMove` prop; the hover-tracking mouse
  events are only exposed on the parent chart container (`<LineChart>`), not on `<Tooltip>`.
  **Fix:** moved `onMouseMove` / `onMouseLeave` up to `<LineChart>` and read `state.activeLabel`
  from there.

- **Symptom:** TS type mismatch on a custom Recharts `Tooltip` `formatter` prop —
  `(value: number, name: string) => [number, string]` didn't satisfy Recharts' expected `Formatter`
  signature built around `ValueType`.
  **Cause:** Recharts' `ValueType` is a broader union (`number | string | Array<number | string>`)
  than the narrower `(value: number, name: string)` signature that was written, so the function type
  didn't structurally match what the `formatter` prop expects.
  **Fix:** dropped the custom formatter and used Recharts' default value formatting, which was
  sufficient for this chart — not worth fighting the generic type for a cosmetic gain.

- **Symptom:** TS2322 pushing `{ from: rangeStart, to: ... }` into an `Array<{ from: string; to: string }>`
  when `rangeStart` is typed `string | null`.
  **Cause:** the null-narrowing of `rangeStart` from an earlier `if` branch didn't carry through to
  every later reference in the same function body, so the compiler kept the union type at the push
  call site.
  **Fix:** added an explicit `rangeStart !== null` check directly in the conditional guarding each
  push.

- **Symptom:** two pre-existing TS errors surfaced in `apps/web/src/components/gis/AppMapCanvas.tsx`
  (image-source coordinates tuple typing) during the `tsc -b --noEmit` pass.
  **Cause (confirmed via evidence, not inferred):** `git status` at the start of this session already
  showed `AppMapCanvas.tsx` as modified outside this worktree's scope — these errors predate this
  session's changes and belong to unrelated GIS work happening in a sibling worktree.
  **Fix:** left untouched; out of scope for this task.

- **Symptom:** `agent-browser screenshot -o <path>` failed during visual QA.
  **Cause:** that flag syntax was assumed from generic CLI conventions, not verified against this
  tool's actual interface; the real CLI takes the output path as a bare positional argument.
  **Fix:** `agent-browser screenshot <path>`.

- **Symptom:** `agent-browser open <url> --width --height` had no effect on viewport size.
  **Cause:** those flags don't exist on `open`; viewport is a separate, namespaced command.
  **Fix:** `agent-browser set viewport <w> <h>` (note: a top-level `agent-browser viewport ...` is
  also not valid — it must go through `set`).

- **Symptom:** `find text "<label>" click` clicked the wrong element (a table `rowheader` with the
  same visible text as an intended metrics-grid button).
  **Cause:** the text-based locator matched the first element with matching text, and the metrics
  grid legitimately reused a label that also appears as a table row header elsewhere on the page.
  **Fix:** switched to `agent-browser snapshot -i` to get a stable `@ref` for the actual intended
  button and clicked that reference directly.

## 8. Verify

```bash
cd apps/web && pnpm exec tsc -b --noEmit
```
Passing output: no errors reported for any `operation-log` component or `OperationLogPage.tsx`/`.css`
(the two `AppMapCanvas.tsx` errors are pre-existing and out of scope, confirmed unrelated to this
change via `git status` at session start).

Visual/interaction QA (manual, via `agent-browser`): loaded the page at 1920px and below 1400px,
confirmed the workspace grid collapses to one column, confirmed clicking a metric card switches the
active chart tab, confirmed hovering the chart updates the comparison table's timestamp label, and
confirmed clicking an event's "Xem chi tiết" opens the detail drawer.

## 9. Gotchas

- `margin: -20px -24px -40px` / `height: calc(100% + 60px)` in `OperationLogPage.css` are hardcoded
  to match `AppShell.css`'s `--pad-page: 20px 24px 40px`. If `--pad-page` ever changes, these values
  will silently go stale — `calc(var(--pad-page) * -1)` cannot be used as a substitute because
  `calc()` only operates on single values, not a multi-value shorthand custom property. Any future
  change to `--pad-page` must be mirrored here by hand.
- The `.ops-dashboard` dark theme only applies within that root class — any new component dropped
  into this page that doesn't nest under `.ops-dashboard` will render with the light theme's
  computed styles instead.
- `mock-data.ts` is a placeholder for SCADA telemetry, documents, decision, and chart series. Once
  real backend endpoints exist, every consumer of `CHART_SERIES_DATA`, `COMPARISON_ROWS`,
  `DECISION`, `DOCUMENTS`, `EVENTS`, and `LIVE_METRICS` needs updating in the same pass — they're
  currently imported by name across multiple components, not behind a single service abstraction.
- `scadaConnection` state exists but nothing currently calls `setScadaConnection` — the
  stale/lost banners are wired but effectively dead code until a real connection-status source is
  plumbed in.
- The temporary `/_preview-operation-log` unauthenticated route used for local QA was removed before
  commit; if a future session needs the same kind of unauthenticated visual QA, it will need to be
  re-added and re-removed the same way, since there's no permanent dev/mock login bypass in
  `AuthContext`/`authService`.
