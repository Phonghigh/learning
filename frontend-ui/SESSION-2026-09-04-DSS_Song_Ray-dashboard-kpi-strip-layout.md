# Session 2026-09-04 — Dashboard KPI row: from 4 equal cards to one asymmetric strip

Branch: `worktree-dashboard-kpi-strip-layout` (pushed to origin, not yet merged). One commit:
`10dc72a feat(web): redesign dashboard KPI row as a single asymmetric summary strip`.

Builds directly on the prior session's work, already logged as
`SESSION-2026-09-03-dashboard-kpi-responsive-redesign.md` (mirrored to
`E:\Learning\frontend-ui\SESSION-2026-09-03-DSS_Song_Ray-dashboard-kpi-responsive-redesign.md`),
which merged 8 single-metric KPI cards into 4 business-grouped cards at equal 25% column widths.

## 1. Requirement recap

The prior session's equal-width 4-card grid was wrong because the four groups carry very different
amounts of content: Mực nước and Dung tích each show one metric, Dòng chảy shows two, Mưa & dự báo
shows four (two visible values plus two in a caption line). Equal columns left the light cards with
dead space and the heavy card cramped. The user (reviewing the merged result) provided a detailed,
self-authored redesign spec proposing:

1. Asymmetric column widths weighted by content amount (example ratio: 0.75fr / 0.75fr / 1.1fr / 1.4fr).
2. Reduced card height (~88-96px) with the caption moved onto the same row as the headline value
   instead of sitting on its own line at the bottom.
3. Merging the four separate bordered/shadowed card boxes into ONE strip container with vertical
   dividers between segments, instead of four distinct "card" boxes.
4. A responsive table: full asymmetric strip ≥1400px, hide caption/secondary info at 1050-1399px,
   compact-but-still-4-across at 800-1049px, 2x2 below 800px.
5. Optionally add sparklines/progress-bars to the water-level and storage cards, but only if real
   historical data exists to back them.

User confirmed via `AskUserQuestion`: implement immediately (not just discuss), and attempt the
sparkline/progress-bar addition only if a real data source exists, otherwise skip.

## 2. How it was implemented + docs used

1. **Checked for a real water-level/storage history API before touching sparklines.** Grepped
   `apps/web/src/services/` — no history endpoint. Read
   `apps/web/src/components/dashboard/WaterLevelChartCard.tsx`, whose own top-of-file comment states:
   "No water-level history endpoint exists yet, so the trend is a demo series anchored to the real
   current reading" (`buildDemoSeries`). Decision: **skip the sparkline/progress-bar entirely.**
   Project rules explicitly forbid mocking implementations ("DO NOT just simulate the implementation
   or mocking them, always implement the real code"), and a KPI sparkline would have meant reusing
   that same fake demo series. Reported to the user as an explicit scope decision, not silently
   dropped — also stated in the commit message.
2. **Restructured `KpiCard.tsx`/`.css`.** Value and caption used to be two separate grid children
   (`grid-template-areas: "icon title" "icon value" "caption caption"`, caption spanning full width
   below). Rewrote to nest value + caption together in a new `.kpi-card-value-row` using
   `display: grid; grid-auto-flow: column` — deliberately **not** `display: flex`, per the standing
   project rule (recorded in user memory) that DSS Song Ray's frontend uses CSS Grid everywhere and
   never Flexbox, even though the user's own written proposal used flex code snippets as illustration.
   Same restructuring applied to `KpiGroupCard.tsx`/`.css` (wrapped title + metrics + caption in a new
   `.kpi-group-card-content` div).
3. **Rewrote `apps/web/src/pages/dashboard-kpi-row.css`** (created in the prior session) from a plain
   1/2/4-equal-column container-query grid into a single "strip": `.dashboard-kpi-row` itself now
   carries the card-like chrome (background/border/border-radius/box-shadow, matching the project's
   shared `.card` utility class) instead of each of the four children owning its own border+shadow.
   Added descendant-selector overrides so `.kpi-card`/`.kpi-group-card` render as flush, borderless
   segments inside the strip, with divider borders (`border-top`/`border-left`) placed per
   container-width tier:
   - **1-col stacked** (<560px): divider = `border-top` between stacked children.
   - **2x2** (560-899px): divider = `border-right` on odd children (vertical split) + `border-top` on
     row 2 (horizontal split).
   - **4-col asymmetric** (≥900px): `grid-template-columns: minmax(150px,0.8fr) minmax(150px,0.8fr)
     minmax(200px,1.2fr) minmax(240px,1.6fr)`, divider = `border-left` on every child but the first.
   - Captions hidden at 900-1199px (not enough room for both the value and caption); restored ≥1200px.
4. **Verified** with `pnpm --filter web build` (`tsc -b && vite build`) after the CSS/component
   restructure, again after `pnpm --filter web exec prettier --write` on the five changed files.
   Grepped the changed CSS for `display:\s*flex` — zero matches, confirming the Grid-only rule was
   honored.
5. Committed as a single commit and pushed to `origin` (worktree/branch pattern already established
   in prior sessions on this repo).

### Options considered / rejected

- **Pure `:not(:last-child)` divider CSS** — rejected: breaks silently once the grid wraps to a new
  row, because the last item of a wrapped row would still show a right divider it shouldn't have, and
  the first item of the new row wouldn't get the top divider it needs. Used explicit `nth-child`
  targeting scoped per container-width tier instead — each tier fully owns its own divider placement
  for its exact column count, with no cross-tier leakage.
- **A `flush`/`strip` modifier prop on `KpiCard`/`KpiGroupCard`** so the components could self-report
  "I'm inside a strip, drop my own chrome" — rejected in favor of scoping the override entirely in
  `dashboard-kpi-row.css` via `.dashboard-kpi-row .kpi-card { ... }` descendant selectors. Keeps the
  components container-agnostic; they could be dropped into a non-strip layout elsewhere without a
  prop threaded through every call site.
- **Building the sparkline anyway using `WaterLevelChartCard`'s existing demo-series fallback** —
  rejected per the explicit no-mock-data project rule; flagged to the user as needing a real backend
  history endpoint first, out of scope for this session.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/web/src/components/dashboard/KpiCard.tsx` | Single-metric KPI card | value and caption were two separate top-level elements (`grid-area: value` / `grid-area: caption`, stacked rows), root `<div>` had class `kpi-card card` | value + caption wrapped together in `.kpi-card-content` > `.kpi-card-value-row` (`grid-auto-flow: column`), caption reads inline next to the number; root `<div>` dropped the shared `card` class |
| `apps/web/src/components/dashboard/KpiCard.css` | Card layout/style | `grid-template-areas: "icon title" "icon value" "caption caption"`, `border: 1px solid #dfe1f7`, padding on the card itself | flat `auto minmax(0,1fr)` grid, no template-areas, no own border (border now lives on the strip wrapper in `dashboard-kpi-row.css`), title/caption get `text-overflow: ellipsis` for the tighter row |
| `apps/web/src/components/dashboard/KpiGroupCard.tsx` / `.css` | Multi-metric KPI card | same stacked-areas pattern as KpiCard, own `border` + `card` class | same content-wrapper restructure (`.kpi-group-card-content`), border and `card` class removed |
| `apps/web/src/pages/dashboard-kpi-row.css` | KPI row container layout | `.dashboard-kpi-row` was a plain grid with `gap: var(--sp-2)`, 1/2/4 equal columns, no visual chrome of its own | `.dashboard-kpi-row` itself IS the visual card (background/border/border-radius/box-shadow); children render flush with divider borders between them; columns go asymmetric fr-widths (`0.8fr 0.8fr 1.2fr 1.6fr`) at the 900px+ tier; captions hidden 900-1199px, restored ≥1200px |

## 4. Code changes in detail

### 1. Strip container chrome + asymmetric columns + tiered dividers — `apps/web/src/pages/dashboard-kpi-row.css`

**Before:**
```css
.dashboard-kpi-row {
  display: grid;
  grid-template-columns: minmax(0, 1fr);
  gap: var(--sp-2);
}

@container kpi-row (min-width: 500px) {
  .dashboard-kpi-row {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

@container kpi-row (min-width: 900px) {
  .dashboard-kpi-row {
    grid-template-columns: repeat(4, minmax(0, 1fr));
  }
}
```

**After (key hunks):**
```css
.dashboard-kpi-row {
  display: grid;
  grid-template-columns: minmax(0, 1fr);
  background: var(--bg-surface);
  border: 1px solid var(--border);
  border-radius: var(--r-md);
  box-shadow: var(--elev-2);
}

.dashboard-kpi-row .kpi-card,
.dashboard-kpi-row .kpi-group-card {
  padding: var(--sp-3);
  min-height: 72px;
  border: none;
  border-radius: 0;
  box-shadow: none;
  background: transparent;
}

.dashboard-kpi-row .kpi-card + .kpi-card,
/* ...+3 more sibling-combinator variants for kpi-card/kpi-group-card pairs */ {
  border-top: 1px solid var(--border);
}

@container kpi-row (min-width: 560px) {
  .dashboard-kpi-row { grid-template-columns: repeat(2, minmax(0, 1fr)); }
  /* border-top rules from the 1-col tier cancelled via matching nth-child selectors,
     not a blanket override — see section 7 */
  .dashboard-kpi-row > *:nth-child(odd) { border-right: 1px solid var(--border); }
  .dashboard-kpi-row > *:nth-child(3) { border-top: 1px solid var(--border); }
  .dashboard-kpi-row > *:nth-child(4) { border-top: 1px solid var(--border); }
}

@container kpi-row (min-width: 900px) {
  .dashboard-kpi-row {
    grid-template-columns:
      minmax(150px, 0.8fr)
      minmax(150px, 0.8fr)
      minmax(200px, 1.2fr)
      minmax(240px, 1.6fr);
  }
  .dashboard-kpi-row > *:nth-child(odd) { border-right: none; border-top: none; }
  .dashboard-kpi-row > *:nth-child(3), .dashboard-kpi-row > *:nth-child(4) { border-top: none; }
  .dashboard-kpi-row > *:not(:first-child) { border-left: 1px solid var(--border); }
  .dashboard-kpi-row .kpi-card-caption,
  .dashboard-kpi-row .kpi-group-card-caption { display: none; }
}

@container kpi-row (min-width: 1200px) {
  .dashboard-kpi-row .kpi-card,
  .dashboard-kpi-row .kpi-group-card { min-height: 88px; }
  .dashboard-kpi-row .kpi-card-caption,
  .dashboard-kpi-row .kpi-group-card-caption { display: block; }
}
```

**What changed:** the row went from a plain `gap`-separated equal-width grid to a chrome-carrying
strip with per-tier asymmetric column widths and divider borders drawn via sibling/`nth-child`
selectors instead of `gap`; a new 900px tier hides captions, a new 1200px tier restores them and
grows `min-height`.

**Why:** solves the dead-space/cramped-card symptom described in the requirement — light segments
(single metric) now get a narrow `0.8fr` track, the heavy segment (Mưa & dự báo, 4 data points) gets
`1.6fr`; merging the four bordered boxes into one strip with internal dividers matches the "one strip,
not four cards" part of the spec.

**How it behaves now:** below 560px the row is one stacked column with horizontal dividers; 560-899px
is a 2x2 grid with a vertical divider down the middle and a horizontal divider between rows; ≥900px is
a single 4-column row with proportionally wider tracks for content-heavier segments and vertical
dividers between all four; captions disappear in the 900-1199px sub-range only, to keep the compact
tier from overflowing, then reappear once there's more room.

### 2. Value + caption on one row — `apps/web/src/components/dashboard/KpiCard.tsx` / `.css`

**Before (`.tsx`):**
```tsx
<div className="kpi-card card">
  {icon && <span className={`kpi-card-icon icon-badge icon-badge--${tone}`}>{icon}</span>}
  <span className="kpi-card-title" title={title}>{title}</span>
  {result.dataAvailable ? (
    <span className="kpi-card-value">
      {displayValue}
      <span className="kpi-card-unit">{unit}</span>
    </span>
  ) : (
    <span className="t-caption kpi-card-no-data">Chưa có dữ liệu</span>
  )}
  {caption && <div className="kpi-card-caption">{caption}</div>}
</div>
```

**After (`.tsx`):**
```tsx
<div className="kpi-card">
  {icon && <span className={`kpi-card-icon icon-badge icon-badge--${tone}`}>{icon}</span>}
  <div className="kpi-card-content">
    <span className="kpi-card-title" title={title}>{title}</span>
    <div className="kpi-card-value-row">
      {result.dataAvailable ? (
        <span className="kpi-card-value">
          {displayValue}
          <span className="kpi-card-unit">{unit}</span>
        </span>
      ) : (
        <span className="t-caption kpi-card-no-data">Chưa có dữ liệu</span>
      )}
      {caption && <span className="kpi-card-caption">{caption}</span>}
    </div>
  </div>
</div>
```

**Before (`.css`):**
```css
.kpi-card {
  padding: var(--sp-2) var(--sp-3);
  display: grid;
  grid-template-columns: auto 1fr;
  grid-template-areas:
    "icon title"
    "icon value"
    "caption caption";
  align-items: center;
  column-gap: var(--sp-2);
  row-gap: 1px;
  border: 1px solid #dfe1f7;
}
```

**After (`.css`):**
```css
.kpi-card {
  display: grid;
  grid-template-columns: auto minmax(0, 1fr);
  align-items: center;
  column-gap: var(--sp-3);
  min-width: 0;
}

/* Value + caption sit on one row (Grid, not Flexbox — project convention)
   so supporting text ("↓ 0.01 m so với hôm qua") reads next to the
   headline number instead of as a separate footer line at the bottom
   of the card. */
.kpi-card-value-row {
  display: grid;
  grid-auto-flow: column;
  justify-content: start;
  align-items: baseline;
  column-gap: var(--sp-2);
  min-width: 0;
}
```

**What changed:** `grid-template-areas` layout replaced with a plain two-column `auto minmax(0,1fr)`
grid; `caption` moved from a full-width `<div>` at grid-area `caption` to a `<span>` inside the new
`.kpi-card-value-row`, which lays value and caption out side-by-side via `grid-auto-flow: column`; the
card's own `border`/`padding` were removed (moved to `dashboard-kpi-row.css`'s strip chrome); title and
caption gained `overflow: hidden; text-overflow: ellipsis; white-space: nowrap` since they now share
less vertical room. `KpiGroupCard.tsx`/`.css` received the identical restructuring
(`.kpi-group-card-content` wrapper around title + metrics + caption).

**Why:** implements requirement point 2 (caption on the same row as the value, reduced card height)
and point 3 (card no longer owns its own border — the strip does).

**How it behaves now:** each card renders shorter (no dedicated caption row eating vertical space),
with the supporting delta text reading inline next to the headline number instead of underneath it.

## 5. How to find this again

Grep terms: `kpi-card-value-row`, `kpi-group-card-content`, `dashboard-kpi-row`, `border-left: 1px
solid var(--border)` (asymmetric-tier divider), `nth-child(odd)` (2x2-tier divider). Route: Dashboard
(`/`). Files: `apps/web/src/components/dashboard/KpiCard.{tsx,css}`,
`apps/web/src/components/dashboard/KpiGroupCard.{tsx,css}`,
`apps/web/src/pages/dashboard-kpi-row.css`.

## 6. Concepts introduced

- **CSS Grid `grid-auto-flow: column`** as the Grid-only substitute for what would normally be
  `display: flex` inline layout — placing a value and its caption side-by-side on one baseline.
  Needed specifically because this project's standing convention forbids Flexbox everywhere, even
  though the user's own written spec used flex snippets as illustration.
- **Divider borders via sibling/`nth-child` selectors, scoped per container-query tier**, as an
  alternative to `gap` + background stripes for a segmented-strip visual. The key reason a blanket
  `:not(:last-child)` rule doesn't work here: divider position depends on where an item sits *within
  its wrapped row* (right edge of the row vs. not), not just its DOM order — a distinction that only
  matters once a grid wraps to multiple rows, which this layout does at the 2x2 tier.

## 7. Where it got stuck

No debugging loop was needed — `pnpm --filter web build` (`tsc -b && vite build`) passed clean on the
first full build after the restructure, and no runtime errors surfaced.

One real design snag did occur during CSS drafting, caught in self-review rather than by a build
error: the initial draft of the 900px (asymmetric 4-col) tier tried to cancel the 2x2 tier's
`border-top` rule with `.dashboard-kpi-row > * { border-top: none !important; }` — a blanket
universal-child selector.
- **Symptom:** using `!important` to force the override worked, but was a red flag on review — it's
  usually a sign a simpler selector-specificity fix exists.
- **Cause:** the 900px tier's `> *` selector has CSS specificity `(0,0,1)`, which is structurally
  *weaker* than the 560px tier's `nth-child(3)`/`nth-child(4)` selectors at `(0,2,1)` it needed to
  override — normal cascade rules alone couldn't make the later rule win without `!important`
  papering over the specificity gap.
- **Fix:** rewrote the 900px tier to target the exact same `nth-child(3)`/`nth-child(4)` selectors
  (matching specificity, later in source order) instead of a blanket `> *`, so plain source-order
  cascade resolved it with no `!important` needed.

## 8. Verify

- `pnpm --filter web build` (runs `tsc -b && vite build`) — passed clean, no type errors, no new
  build warnings (only the pre-existing, unrelated "chunk >500kB" bundle-size notice).
- Grepped the five changed files for `display:\s*flex` — zero matches, confirming the Grid-only
  convention was honored throughout.
- No automated browser testing was performed, per this repo's convention of relying on `tsc`/build
  plus user screenshots rather than Claude-driven browser testing. **User should visually verify** at
  the tier boundaries: <560px (1-col stacked), 560-899px (2x2), 900-1199px (4-col, captions hidden —
  confirm it doesn't read as broken/incomplete), ≥1200px (4-col, captions shown, asymmetric ratios
  readable).

## 9. Gotchas

- The strip's border/shadow now lives on `.dashboard-kpi-row` itself, not on the individual
  `.kpi-card`/`.kpi-group-card` elements. If either component is ever reused **outside** this row
  (e.g. a future standalone single-KPI widget elsewhere), it will render borderless/flush by default,
  because the "own card chrome" was stripped only via the `.dashboard-kpi-row .kpi-card` descendant
  selector — check whether that scoping still applies before assuming a standalone `KpiCard` looks
  right elsewhere.
- The divider CSS is manually tier-matched to exactly **4 children**. If a 5th KPI group is ever added
  to this row, both the `nth-child(3)`/`nth-child(4)` divider rules in the 560px tier and the
  asymmetric column-width list in the 900px tier need updating by hand — nothing here generalizes
  automatically to N children.
- Captions disappear entirely (not truncate) at the 900-1199px container-width tier — if
  `waterLevelDelta`, `storagePercent`, or the rainfall/forecast caption line ever carry
  safety-relevant information, that tier silently hides it. This mirrors a similar tradeoff already
  flagged in the prior session's report for the 8-card layout.
- Sparkline/progress-bar for water level and storage is **not implemented** — explicitly deferred
  pending a real history API (see section 2). Don't assume it was simply forgotten if revisiting this
  area; check for a real `/water-level/history`-style endpoint first.
