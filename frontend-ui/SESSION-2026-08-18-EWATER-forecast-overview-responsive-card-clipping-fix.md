<!--
  LEARN-LOG REPORT
  Ad-hoc bug fix, no tracked task id — see CLAUDE.md learn-log rule for the exception.
-->

# forecast-overview-responsive-card-clipping-fix — `/forecast/overview` charts collapse to a sliver below 1200px

**Date:** 2026-08-18

---

## 1. Requirement recap

User report (with a screenshot) on `/forecast/overview`: below the 1200px viewport-width breakpoint,
the "Dự báo mưa" (rain forecast) and "Dự báo mực nước" (water-level forecast) chart cards lose their
content — only a truncated top axis label is visible ("2mm-", "1.2m-"), with the rest of each chart
invisible. Ad-hoc bug fix reported during the session — not tied to a `tasks/INDEX.md` task id.
Same session as the `/gis-map` responsive fix logged separately in
`docs/learn-log/gis-map-responsive-height-clamp-fix.md` — this is a different page/bug, kept as its
own report per instruction.

## 2. How it was implemented + docs used

`.fc-top-grid` (`web/src/styles.css`) is a 3-column × 3-row CSS grid on desktop with a fixed
`height: clamp(640px, calc(100vh - 130px), 1300px)`, holding 4 cards: `.fc-flood-card` (mini flood
map, spans 2 cols × 3 rows), `.fc-result-card` (forecast result stats), and two `.dash-chart-card`s
(rain forecast; water-level forecast, which also carries `.fc-wlf-card`). Each chart card contains a
`.dash-chart-canvas` (`flex: 1; height: auto` inside a `display:flex; flex-direction:column` card)
wrapping a Recharts `<ResponsiveContainer width="100%" height="100%">` (`RainForecastChart.tsx`, and
the reused water-level chart component).

The existing `@media (max-width: 1200px)` override switched the grid to a single stacked column
(`grid-template-columns: 1fr; grid-template-rows: none;`) so the 4 cards stack vertically in DOM
order, but it did **not** touch `.fc-top-grid`'s own `height`, which stayed pinned to the desktop's
fixed clamp. Four stacked cards' natural content height is much larger than that fixed box, so
content overflowed/got clipped instead of the page growing to fit and scrolling.

This is the same class of bug already documented and fixed elsewhere in this stylesheet for
`.mon-page-grid`/`.gis-page-grid` (see the sibling report on `/gis-map` from this same session): a
flex/grid child with `flex: 1; height: auto` collapses toward 0 when its ancestor chain doesn't
resolve to a real pixel height, and Recharts' `ResponsiveContainer height="100%"` needs exactly that
real height to compute from — with none available it renders essentially nothing but the topmost
axis gridline label, which matches the screenshot exactly.

**Fix, reusing the pattern already established for `.dash-main-row`'s own `≤1200px` override**
(`.dash-side2 .dash-chart-canvas, .dash-side3 .dash-chart-canvas { flex: none; height: 190px; }`):
give `.fc-top-grid` an `auto` height so the stacked column grows to fit its content and the page
scrolls, and give the two chart canvases an explicit pixel height at this breakpoint instead of
depending on flex-fill. 220px chosen by eyeballing consistency with the existing 190px precedent,
scaled up slightly since this chart's card header carries more chrome (axis labels + dropdowns).

Existing code reused: the `.dash-side2`/`.dash-side3` explicit-height pattern. No new CSS
primitives, tokens, or files needed.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `web/src/styles.css` (`@media (max-width: 1200px)` block, `.fc-top-grid`, ~line 1858) | Responsive grid rule for `/forecast/overview`'s 4-card top section (mini map, result stats, rain forecast, water-level forecast) | `grid-template-columns: 1fr; grid-template-rows: none;` — kept the desktop fixed `height: clamp(640px, ..., 1300px)` from the base rule, cards stacked but clipped by that fixed box | Added `height: auto;` on `.fc-top-grid`, plus `.fc-top-grid > .dash-chart-card { flex: none; }` and `.fc-top-grid > .dash-chart-card .dash-chart-canvas { flex: none; height: 220px; }` — grid grows to real content height, both chart canvases get an explicit pixel height for Recharts to render into |

## 4. Code changes in detail

### 1. `.fc-top-grid` gains `height: auto` at `≤1200px` — `web/src/styles.css`

**Before:**
```css
@media (max-width: 1200px) {
  /* Dưới 1200px: 1 cột, mọi card trở về thứ tự tự nhiên trong DOM (map, kết
     quả, mưa, mực nước dự báo) thay vì lưới 3 cột × 3 track. */
  .fc-top-grid { grid-template-columns: 1fr; grid-template-rows: none; }
  .fc-top-grid > .fc-flood-card,
  .fc-top-grid > .fc-result-card,
  .fc-top-grid > .dash-chart-card {
    grid-column: 1; grid-row: auto;
  }
}
```

**After:**
```css
@media (max-width: 1200px) {
  /* Dưới 1200px: 1 cột, mọi card trở về thứ tự tự nhiên trong DOM (map, kết
     quả, mưa, mực nước dự báo) thay vì lưới 3 cột × 3 track.
     2026-08-18 fix ("<1200 /forecast/overview đang chưa có responsive, các
     card đang bị mất thông tin"): base rule để `height: clamp(640px, ...,
     1300px)` trên CHÍNH `.fc-top-grid` — ở 1 cột 4 card xếp dọc, tổng chiều
     cao nội dung thật lớn hơn hẳn khung cố định đó, `grid-template-rows:
     none` (auto) không co giãn được để bù nên card cuối bị cắt/tràn. Đổi
     sang `height: auto` để cả khối cao theo đúng nội dung, trang tự cuộn —
     cùng công thức đã dùng cho `.gis-page-grid`/`.mon-page-grid` ở breakpoint
     này. */
  .fc-top-grid { grid-template-columns: 1fr; grid-template-rows: none; height: auto; }
  .fc-top-grid > .fc-flood-card,
  .fc-top-grid > .fc-result-card,
  .fc-top-grid > .dash-chart-card {
    grid-column: 1; grid-row: auto;
  }
  /* Hệ quả của `height: auto` ở trên: `.dash-chart-card` (Dự báo mưa/Mực
     nước dự báo) không còn chiều cao THẬT từ grid row để `.dash-chart-canvas`
     (`flex: 1; height: auto`) tính theo — Recharts `ResponsiveContainer
     height="100%"` cần cha có px thật, không có thì co về ~0 (chỉ còn nhãn
     trục trên cùng, đúng ảnh chụp lỗi) — cùng bug "flex:1 auto collapse" đã
     gặp ở MapLibre (xem `.gis-page-grid`/`.mon-page-grid` comments). Gán
     chiều cao TƯỜNG MINH cho canvas ở breakpoint này, cùng khuôn với
     `.dash-side2 .dash-chart-canvas`/`.dash-side3 .dash-chart-canvas`
     (`.dash-main-row`'s ≤1200px override, xem phía trên). */
  .fc-top-grid > .dash-chart-card { flex: none; }
  .fc-top-grid > .dash-chart-card .dash-chart-canvas { flex: none; height: 220px; }
}
```

**What changed:** `.fc-top-grid` gained `height: auto` (was implicitly still the desktop fixed
clamp, since the override never set `height` at all). Two new rules were added below the existing
column/row reset: `.fc-top-grid > .dash-chart-card { flex: none; }` and
`.fc-top-grid > .dash-chart-card .dash-chart-canvas { flex: none; height: 220px; }`. Comment above
the block updated with the 2026-08-18 date, the user's verbatim complaint, and an explicit
explanation tying it to the same MapLibre/Recharts collapse pattern already documented for
`.gis-page-grid`/`.mon-page-grid`.

**Why:** without `height: auto`, the stacked single-column layout was still forced into the
desktop's fixed `clamp(640px, ..., 1300px)` box, so content past that box was clipped instead of the
page growing/scrolling. Separately, once the grid rows became `none`/auto, `.dash-chart-card` no
longer handed its `.dash-chart-canvas` a real pixel height through the grid track — and
`flex: 1; height: auto` on the canvas has nothing to resolve against without one, so Recharts'
`ResponsiveContainer height="100%"` collapsed toward 0, leaving only the topmost axis label visible
(matching the screenshot: "2mm-", "1.2m-" slivers).

**How it behaves now:** at widths ≤1200px, `.fc-top-grid` grows to its stacked content's real
height and the page scrolls normally instead of clipping; both `.dash-chart-card` instances (rain
forecast, water-level forecast) get a fixed 220px canvas height, giving `ResponsiveContainer` a real
ancestor height to render the chart into instead of collapsing.

**Not touched:** `.fc-flood-card`'s mini map — it already used an explicit `min-height: 360px` on
`.fc-flood-map` rather than relying on flex-fill, so it was unaffected by this bug and rendered
correctly in the user's screenshot.

## 5. How to find this again

- `grep -n "fc-top-grid" web/src/styles.css` — base rule ~line 1810, `≤1200px` override ~line 1856.
- `grep -n "dash-chart-canvas" web/src/styles.css` — the same collapse pattern and its existing fix
  precedent (`.dash-side2`/`.dash-side3` at 190px inside `.dash-main-row`'s own override).
- Route: `/forecast/overview`. Components: `web/src/components/RainForecastChart.tsx`,
  `web/src/components/gis/GisWaterLevelForecastChart.tsx` (reused for the water-level card here).

## 6. Concepts introduced

### Recharts `ResponsiveContainer height="100%"` needs a real ancestor pixel height (referenced, not new)
- **Plain definition:** `ResponsiveContainer` measures its parent element's rendered size to decide
  the chart's pixel dimensions; if that parent's height itself depends on `height: auto`/`flex: 1`
  with nothing above it resolving to a real number, the container measures ~0 and renders almost
  nothing.
- **Why it shows up here:** this is the exact same bug already documented in this stylesheet for
  MapLibre canvases on `/gis-map` and `/monitoring` (this session's other fix). Recognizing the
  pattern meant reaching straight for the established fix (explicit pixel height at the breakpoint)
  instead of re-diagnosing from scratch.

## 7. Where it got stuck

- **Symptom:** user screenshot showed both chart cards reduced to a thin sliver containing only the
  topmost axis gridline label ("2mm-", "1.2m-"), rest of the chart area blank.
- **Root cause (INFERRED, not confirmed by rendering in a browser):** read through `.fc-top-grid`'s
  base rule and its `≤1200px` override side by side and found the override never set `height`, so
  the grid stayed pinned to the desktop's fixed clamp while switching to a taller stacked layout —
  directly readable from the CSS diff, no ambiguity there. The *Recharts-collapsing-to-0* mechanism
  specifically (rather than some other clipping reason) was reasoned by analogy to the already-fixed
  and documented `.gis-page-grid`/`.mon-page-grid` bug in the same file, plus the shape of
  `.dash-chart-canvas`'s `flex: 1; height: auto` and `RainForecastChart.tsx`'s
  `ResponsiveContainer height="100%"` matching that known-bad pattern exactly. It was **not**
  independently confirmed by opening dev tools or resizing a real browser this session — the
  screenshot's symptom (only the axis label visible) is consistent with this mechanism but wasn't
  directly instrumented to prove it.
- **Second finding worth flagging going forward:** this is the second page in the same session
  (after `/gis-map`) found with the identical anti-pattern class — a `@media (max-width: 1200px)`
  override that changes a grid's column/row structure but forgets to also relax the container's
  fixed `height`, combined with a `flex: 1; height: auto` child depending on that fixed height to
  render. Worth grepping `styles.css` for other `@media` blocks touching `*-page-grid`/`*-top-grid`
  containers if more responsive complaints come in — this may not be the last instance.

## 8. Verify

```bash
cd "E:\FRIMS_VINH_LONG\EWATER" && node -e "
const s = require('fs').readFileSync('web/src/styles.css','utf8');
let d = 0; for (const c of s) { if (c==='{') d++; if (c==='}') d--; }
console.log('brace balance:', d);
"
grep -n "fc-top-grid" web/src/styles.css
```
Expected: brace balance `0` (no syntax break introduced), and the grep output shows the
`≤1200px` block now including `height: auto;` on `.fc-top-grid` and the two new
`.dash-chart-card`/`.dash-chart-canvas` rules.

**Not yet verified:** this session did **not** visually confirm the fix in a real browser — the
user has been declining browser automation this session, so there is no screenshot/re-confirmation
that the rain-forecast and water-level-forecast charts actually render fully now (as opposed to just
no longer throwing a CSS syntax error). Recommended follow-up: open `/forecast/overview` in a real
browser, resize below 1200px, and confirm (a) both chart cards render the full chart (axes + lines,
not just the top gridline label), and (b) the page scrolls cleanly through all 4 stacked cards
without any of them clipping.

## 9. Gotchas

- The 220px canvas height is a guess at "looks reasonable," not measured against the chart's actual
  minimum readable size at that width — if a future session adds more series/legend chrome to
  `RainForecastChart.tsx` or the water-level chart, 220px may need raising.
- This fix was not visually verified — treat it as a plausible fix pending real-browser
  confirmation, matching the same caveat on the sibling `/gis-map` report from this session.
- If `.fc-top-grid` ever regains its 3-column desktop-style layout at a narrower breakpoint (instead
  of the current single-column stack), the `flex: none; height: 220px;` override on the chart canvas
  would need to move or be conditioned on the column count — it currently assumes the stacked,
  single-column shape.
