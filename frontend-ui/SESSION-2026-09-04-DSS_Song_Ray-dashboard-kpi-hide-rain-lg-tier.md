# Session — Hide two lowest-priority rain KPI cards at the lg breakpoint

**Date:** 2026-09-04
**Commit:** `ed04864` — `worktree-dashboard-kpi-hide-rain-lg-tier`

## 1. Requirement recap

User's literal instruction (Vietnamese): "từ 1200>= x <1536 thì cho card Mưa 24 giờ qua và Mưa dự
báo 72 giờ tới, ẩn đi" — at 1200<=width<1536px, hide the "Mưa 24 giờ qua" (rainfall 24h) and "Mưa dự
báo 72 giờ tới" (forecast 72h) KPI cards on the Dashboard page.

## 2. How it was implemented + docs used

This is the 3rd session addressing the same underlying layout bug: `.dashboard-kpi-row` is a
6-column grid at the `--bp-lg-up` (1200px+) tier but renders 8 `KpiCard`s, leaving a ragged 6+2
two-row layout. The two prior sessions (`SESSION-2026-09-03-dashboard-kpi-responsive-redesign.md`,
`SESSION-2026-09-04-dashboard-kpi-strip-layout.md`) attacked this with progressively bigger
redesigns (container queries + card grouping, then an asymmetric single-strip layout) — both were
fully reverted from `main` because the user decided that scope was more than they wanted.

This session took the opposite approach: instead of changing the grid or the cards, hide 2 of the 8
cards specifically in the 1200-1535px tier so the remaining 6 fill the existing 6-column grid
exactly. All 8 reappear at `--bp-xl-up` (1536px+), where the grid is already 8 columns. No changes
to the shared `KpiCard` component, no new files, no grid restructuring — the smallest fix that
satisfies the literal instruction.

Card choice: "Mưa 6 giờ qua" and "Mưa dự báo 24 giờ tới" stay visible and already cover the same
signal at shorter/nearer horizons than the two hidden ones (24h-actual, 72h-forecast), so hiding
those two loses the least operationally relevant information in that tier.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/web/src/pages/DashboardPage.tsx` | KPI row markup | 8 `KpiCard`s rendered flat in `.dashboard-kpi-row` | Two of them (`rainfall24h`, `forecast72h`) wrapped in `<div className="dashboard-kpi-item--hide-lg">` |
| `apps/web/src/pages/DashboardPage.css` | KPI row responsive rules | 6-col rule at `--bp-lg-up`, no handling for the 8-vs-6 card mismatch | Added `@media (--bp-lg-up) and (--bp-xl-down) { .dashboard-kpi-item--hide-lg { display: none; } }` next to the 6-col rule |

## 4. Code changes in detail

### 1. Wrap the two lower-priority rain cards in a hide-at-lg utility div — `apps/web/src/pages/DashboardPage.tsx`

**Before:**
```tsx
<KpiCard
  title="Mưa 24 giờ qua"
  unit="mm"
  result={kpis?.rainfall24h ?? NO_DATA}
  icon={<CloudRainIcon />}
  tone="gray"
  caption="Tại lưu vực hồ"
/>
```
```tsx
<KpiCard
  title="Mưa dự báo 72 giờ tới"
  unit="mm"
  result={kpis?.forecast72h ?? NO_DATA}
  icon={<CloudRainIcon />}
  tone="purple"
  caption="Dự báo trung hạn"
/>
```

**After:**
```tsx
<div className="dashboard-kpi-item--hide-lg">
  <KpiCard
    title="Mưa 24 giờ qua"
    unit="mm"
    result={kpis?.rainfall24h ?? NO_DATA}
    icon={<CloudRainIcon />}
    tone="gray"
    caption="Tại lưu vực hồ"
  />
</div>
```
```tsx
<div className="dashboard-kpi-item--hide-lg">
  <KpiCard
    title="Mưa dự báo 72 giờ tới"
    unit="mm"
    result={kpis?.forecast72h ?? NO_DATA}
    icon={<CloudRainIcon />}
    tone="purple"
    caption="Dự báo trung hạn"
  />
</div>
```

**What changed:** each of the two `<KpiCard>` calls got wrapped in a plain `<div className="dashboard-kpi-item--hide-lg">`. Nothing inside `KpiCard` itself changed.
**Why:** a CSS class needs a DOM node to target for `display: none`; `KpiCard` has no `className`/wrapper prop, so a thin wrapper div was the smallest way to attach the class without touching the shared component.
**How it behaves now:** at 1200-1535px viewport width the wrapper (and the card inside it) is removed from layout, so the grid's 6 columns are filled by exactly 6 visible cards instead of 6+2.

### 2. Add the hide-at-lg media rule — `apps/web/src/pages/DashboardPage.css`

**Before:** (no rule existed between the lg-up 6-col grid rule and the xl-up 8-col grid rule)

**After:**
```css
/* At 1200-1535px the grid above is 6 cols but there are 8 KPI cards,
   leaving a ragged 6+2 second row. Hiding the two lowest-priority cards
   (Mưa 24 giờ qua, Mưa dự báo 72 giờ tới — the 6h reading and 24h forecast
   next to them already cover the same signal at shorter/nearer horizons)
   makes the remaining 6 fill exactly one row. Reappear at xl-up (8 cols,
   room for all 8). */
@media (--bp-lg-up) and (--bp-xl-down) {
  .dashboard-kpi-item--hide-lg {
    display: none;
  }
}
```

**What changed:** new media rule added, scoped with the existing `--bp-lg-up`/`--bp-xl-down` custom media queries from the project's breakpoints system (not new custom media — reused).
**Why:** confines the hide behavior to exactly the 1200-1535px tier the user specified; outside that range the cards render normally.
**How it behaves now:** below 1200px the cards are visible (grid wraps into more rows anyway at that tier per existing rules); at 1536px+ the 8-col grid rule takes over and all 8 cards show.

*(The commit also includes unrelated prettier reformatting of pre-existing lines in both files — grid-area one-liners expanded to multi-line, a stray blank line removed, some prop-wrapping changes to satisfy the line-length rule. These are formatting-only, not logic changes.)*

## 5. How to find this again

- Class name: `dashboard-kpi-item--hide-lg`
- Media query pair: `--bp-lg-up`, `--bp-xl-down` (defined in the shared breakpoints custom-media file, not this commit)
- Grid container: `.dashboard-kpi-row` in `DashboardPage.css`
- Card keys: `kpis?.rainfall24h`, `kpis?.forecast72h` in `DashboardPage.tsx`

## 6. Concepts introduced

None new — this reuses the container/media-query and custom-media (`--bp-lg-up` etc.) pattern already
established in the project's breakpoints system from prior sessions.

## 7. Where it got stuck

No real snag this session. `pnpm --filter web build` (which runs `tsc` then `vite build`) passed on
the first attempt, and grepping the compiled `dist/assets/*.css` confirmed the media rule compiled
to the expected `(min-width:1200px) and (width<=1535px)` range. Prettier was run on both changed
files before the final build/commit, which is why the diff also touches unrelated pre-existing lines.

Design note (not a snag): this approach was deliberately chosen to be the smallest, most easily
reversible fix in this saga — `display: none` on a wrapper div, zero shared-component changes, zero
new files — specifically because the two prior sessions' bigger redesigns of the same problem were
both reverted for being more than the user wanted.

## 8. Verify

```
pnpm --filter web build
```
Expected: `tsc` and `vite build` both exit 0. Then, to confirm the media query compiled correctly:
```
grep -n "dashboard-kpi-item--hide-lg" apps/web/dist/assets/*.css
```
Expected output shows the rule wrapped in `@media (min-width:1200px) and (width<=1535px){...display:none...}`.

## 9. Gotchas

- `dashboard-kpi-item--hide-lg` is a one-off utility class scoped to this exact page and this exact
  media tier — it is not a general-purpose "hide at lg" utility, despite the generic-sounding name.
- The wrapper div sits inside a CSS Grid row where each direct child is expected to be a grid item.
  The wrapper itself *is* the grid item now (with `KpiCard` as its plain nested child). This only
  works cleanly because `KpiCard` has no fixed width and stretches to fill its parent. If `KpiCard`'s
  CSS ever gets a fixed width, or `.dashboard-kpi-row`'s `align-items`/`justify-items` changes, re-check
  that the wrapper still stretches to fill the grid cell correctly.
- If a 4th session revisits this same 6-vs-8-card mismatch, check this file's history first — three
  different fixes have now been tried for the same underlying grid/card-count mismatch.
