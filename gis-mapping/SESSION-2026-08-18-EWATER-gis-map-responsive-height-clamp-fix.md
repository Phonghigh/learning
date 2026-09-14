<!--
  LEARN-LOG REPORT
  Ad-hoc bug fix, no tracked task id — see CLAUDE.md learn-log rule for the exception.
-->

# gis-map-responsive-height-clamp-fix — `/gis-map` doesn't fill viewport below 1200px, detail-card invisible

**Date:** 2026-08-18

---

## 1. Requirement recap

User report: on the `/gis-map` page, below the 1200px viewport-width breakpoint, the map canvas
does not fill the screen, and the gate/rain-station detail-card (`.gis-side-column`) does not
display properly. Ad-hoc bug fix reported by the user during this session — not tied to a
`tasks/INDEX.md` task id.

## 2. How it was implemented + docs used

No new pattern was introduced; this reused the existing viewport-fill formula already used by the
desktop (`>1536px`) base rule for `.gis-page-grid` (`web/src/styles.css:704-719`):
`grid-template-rows: auto minmax(0, 1fr); height: calc(100vh - 130px);` — a two-row grid
(`topbar` + `toprow`) where `toprow` is the single flexible track that expands to fill whatever
viewport height remains after the fixed 130px of chrome above it (app topbar + page padding).

The two responsive overrides at `max-width: 1200px` and `max-width: 899px` had instead hard-coded
a small `clamp()` cap on that same row (`clamp(360px, 48vh, 560px)` and `clamp(280px, 45vh, 420px)`
respectively) plus `height: auto`. Reading the comment history in the file (git blame via inline
dated comments, a convention already established in this codebase) showed this pattern was
originally written for a *different* stylesheet section — `.mon-page-grid` (the Monitoring page,
~`styles.css:2530`) — which has **five** competing grid tracks (`auto`/`1fr` rows for map + 2
tables + 3 charts). There, letting more than one track resolve via `auto`/`minmax(0,1fr)` caused
tracks to fight for space and collapse toward 0 height, so a fixed clamp was the correct fix for
that page.

`.gis-page-grid` only ever has **one** flexible track (`toprow` — everything else is `auto`), so
the multi-track collapse bug that justified the `.mon-page-grid` clamp does not apply here. The
clamp was carried over from `.mon-page-grid`'s fix by copy-paste (inferred from the near-identical
`clamp()`/`height: auto` shape and adjacent code history, not from a commit message — this
codebase's own comments reference "cùng quyết định đã áp dụng cho `.mon-page-grid`" i.e. "same
decision as applied to `.mon-page-grid`" directly in the pre-fix comment, confirming the copy was
intentional at the time but became stale once the underlying justification (5-track collapse) no
longer matched `.gis-page-grid`'s actual structure).

**Fix:** replace the hard clamp in both overrides with the same `minmax(floor, 1fr)` +
`height: calc(100vh - 130px)` formula the desktop rule already uses, keeping a small pixel floor
(360px / 280px) purely so very short viewports don't collapse toward 0, not to artificially cap the
row on tall viewports.

Existing code reused: the desktop base rule's formula (`styles.css:715,718`). No new CSS
primitives, tokens, or files were needed — pure geometry values inside two existing `@media`
blocks.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `web/src/styles.css` (`@media (max-width: 1200px)` block, `.gis-page-grid`, ~line 2404) | Responsive row-height rule for the GIS map page's single flexible row (`toprow`, holding canvas + `.gis-side-column`) | `grid-template-rows: auto clamp(360px, 48vh, 560px); height: auto;` — hard-capped at 560px regardless of real viewport height | `grid-template-rows: auto minmax(360px, 1fr); height: calc(100vh - 130px);` — fills viewport like desktop, 360px floor only |
| `web/src/styles.css` (`@media (max-width: 899px)` block, `.gis-page-grid`, ~line 2557) | Same rule, narrower breakpoint | `grid-template-rows: auto clamp(280px, 45vh, 420px); height: auto;` — hard-capped at 420px | `grid-template-columns: 1fr; grid-template-rows: auto minmax(280px, 1fr); height: calc(100vh - 130px);` — fills viewport, 280px floor only |

## 4. Code changes in detail

### 1. `<1200px` breakpoint stops capping `.gis-page-grid` row height — `web/src/styles.css`

**Before:**
```css
@media (max-width: 1200px) {
  /* 2026-08-11: `.gis-bottom-row` xoá hẳn (camera dock vào `.gis-side-column`
     cạnh detail panel, xem rule gốc của `.gis-page-grid`) -grid chỉ còn 2
     hàng (topbar + toprow), `height: auto` để TRANG tự cuộn ở màn hẹp/thấp
     nếu nội dung không vừa, cùng quyết định đã áp dụng cho `.mon-page-grid`
     ở các mốc hẹp tương tự. */
  .gis-page-grid {
    grid-template-rows: auto clamp(360px, 48vh, 560px);
    height: auto;
  }
```

**After:**
```css
@media (max-width: 1200px) {
  /* 2026-08-18 fix ("<1200 responsive không show map full màn hình, và
     detail-card trong tab này"): bản trước ép `toprow` vào 1 clamp CỐ ĐỊNH
     (360-560px) bất kể chiều cao viewport thật -map bị nén nhỏ dù còn thừa
     chỗ, và `.gis-side-column` (detail-card gate/trạm mưa + camera, xem rule
     gốc `.gis-side-column`) bị dồn vào cùng khoảng cao nhỏ đó nên hầu như
     không thấy được. `.gis-page-grid` chỉ có 1 hàng co giãn DUY NHẤT
     (`toprow`) -không phải trường hợp nhiều track `1fr` tranh nhau đã gây
     sập về 0 ở `.mon-page-grid` (xem comment "5 track" phía trên) - nên dùng
     lại ĐÚNG công thức lấp viewport của desktop (`minmax(.., 1fr)` +
     `height: calc(100vh - 130px)`), chỉ thêm sàn 360px để không sập hẳn ở
     màn rất thấp. */
  .gis-page-grid {
    grid-template-rows: auto minmax(360px, 1fr);
    height: calc(100vh - 130px);
  }
```

**What changed:** `grid-template-rows`'s second track changed from `clamp(360px, 48vh, 560px)`
(a hard ceiling of 560px) to `minmax(360px, 1fr)` (a floor of 360px, otherwise fully flexible);
`height: auto` (let the page grow/scroll to fit content) changed to
`height: calc(100vh - 130px)` (pin the grid's total height to the viewport, same as desktop).
Comment above updated with the 2026-08-18 date, the user's verbatim complaint, and an explicit
explanation of why `minmax(.., 1fr)` is safe here (single flexible track) vs. why it wasn't safe
for `.mon-page-grid`.

**Why:** the clamp capped `toprow`'s height at 560px even on viewports far taller than that,
starving `.gis-side-column` (which lives inside `toprow` alongside the canvas) of vertical space
and making the detail-card look broken/cut off. `height: auto` compounded this by letting the grid
container shrink to fit the capped row instead of claiming the full available viewport.

**How it behaves now:** at widths between 900px and 1200px, `.gis-page-grid`'s second row expands
to fill `calc(100vh - 130px)` exactly like the desktop layout does, only falling back to the 360px
floor on viewports too short to offer more than that.

### 2. `<899px` breakpoint — same fix, narrower floor — `web/src/styles.css`

**Before:**
```css
  .gis-page-grid {
    grid-template-columns: 1fr;
    grid-template-rows: auto clamp(280px, 45vh, 420px);
    height: auto;
  }
```

**After:**
```css
  .gis-page-grid {
    grid-template-columns: 1fr;
    grid-template-rows: auto minmax(280px, 1fr);
    height: calc(100vh - 130px);
  }
```

**What changed:** identical transformation as change 1, with a 280px floor instead of 360px
(matching the original clamp's lower bound), plus the same comment rewrite explaining the
single-track vs. multi-track distinction with `.mon-page-grid`.

**Why / How it behaves now:** same as change 1, applied to the narrower breakpoint where
`.gis-page-grid` also switches to a single column.

## 5. How to find this again

- `grep -n "gis-page-grid" web/src/styles.css` — base rule at ~line 704, overrides at ~line 2404
  (`max-width: 1200px`) and ~line 2557 (`max-width: 899px`).
- `grep -n "mon-page-grid" web/src/styles.css` — the sibling page whose 5-track collapse bug
  originally motivated the clamp pattern that got miscopied onto `.gis-page-grid`.
- Route: `/gis-map`. Components: `web/src/gis/GisMap.tsx` (or wherever `.gis-side-column`,
  `.gis-top-row`, `.gis-canvas-wrapper` are rendered).

## 6. Concepts introduced

### CSS Grid `minmax()` vs `clamp()` for a flexible track
- **Plain definition:** `minmax(floor, 1fr)` on a grid row means "never smaller than `floor`, but
  otherwise grow to absorb all leftover space in the grid container." `clamp(min, preferred, max)`
  on a row's height means "always resolve to `preferred` (here a `vh` percentage), but never below
  `min` or above `max`" — critically, `clamp()`'s value is fixed relative to the viewport
  percentage, not to how much space is actually left in the parent grid, so it can leave the
  container's true `height` (which was `auto`) larger or smaller than the row actually needs.
- **Why it shows up here:** the bug was exactly this distinction — `clamp(360px, 48vh, 560px)`
  ignores how tall the actual grid container is and just returns a viewport-relative number capped
  at 560px, while `minmax(360px, 1fr)` cooperates with `height: calc(100vh - 130px)` on the parent
  to actually fill whatever space is available.

### Grid track collapse with multiple `1fr`/`auto` rows (referenced, not introduced)
- **Plain definition:** when a grid has several rows sized `auto` or `minmax(0, 1fr)` and the
  combined content can't fit the container, the flexible rows can shrink toward 0 instead of
  growing, because there's no single row absorbing 100% of the slack.
- **Why it shows up here:** this is the *pre-existing* bug documented for `.mon-page-grid` (5
  competing tracks) that the `.gis-page-grid` clamp was copied from. It does not actually apply to
  `.gis-page-grid`, which only has one flexible row — this session's core finding was recognizing
  that the copied fix's justification didn't transfer.

## 7. Where it got stuck

- **Symptom:** map not filling the screen and detail-card not "showing properly" below 1200px.
- **False lead considered:** hypothesized that `.gis-top-row`'s flex layout might need
  `flex-direction: column` at narrow widths so `.gis-side-column` stacks below the canvas instead
  of squeezing it horizontally — prompted by `.gis-side-column`'s `flex: 0 0 clamp(280px, 34%, 380px)`
  having no `min-width` guard, which could in theory force horizontal overflow at very narrow
  widths. Ruled out as the primary cause without pursuing a code change, because the width math
  looked acceptable at the realistic breakpoint range (≥900px, where `.gis-side-column` still gets
  a full row via `grid-template-columns: 1fr` only below 899px) and because the height clamp was
  directly visible and dated in the CSS itself as a stronger, more evidenced candidate. **This
  ruling-out is inferred from static CSS reading, not confirmed by rendering in a browser** — it
  remains a secondary hypothesis that was deprioritized, not disproven.
- **Also checked:** grepped the whole stylesheet for `display: none` or `overflow: hidden` rules
  that might be hiding `.gis-side-column` outright. None found — ruled out.
- **Root cause identified (also inferred, not directly observed in a running browser):** the
  `clamp(...)` + `height: auto` pattern on `.gis-page-grid`'s single flexible row was copy-pasted
  from `.mon-page-grid`'s fix for a 5-track collapse bug that doesn't apply to `.gis-page-grid`
  (only 1 flexible track), causing an unnecessary hard height cap that starved both the map canvas
  and `.gis-side-column` of available vertical space. Evidence: the pre-fix comment in the file
  explicitly states "cùng quyết định đã áp dụng cho `.mon-page-grid`" (same decision applied to
  `.mon-page-grid`), and the structural difference (1 flexible track vs. 5) is directly readable
  from the surrounding CSS rules for each page's grid.

## 8. Verify

```bash
cd "E:\FRIMS_VINH_LONG\EWATER" && node -e "
const s = require('fs').readFileSync('web/src/styles.css','utf8');
let d = 0; for (const c of s) { if (c==='{') d++; if (c==='}') d--; }
console.log('brace balance:', d);
"
grep -n "gis-page-grid" web/src/styles.css
```
Expected: brace balance `0` (no syntax break introduced), and the grep output shows both edited
`@media` blocks now reading `grid-template-rows: auto minmax(...)` with
`height: calc(100vh - 130px)` alongside the original desktop rule.

**Not yet verified:** this session did **not** visually confirm the fix in a real browser — the
user explicitly declined browser automation for this session. Recommended follow-up: open
`/gis-map` in a real browser, resize below 1200px and below 899px, both with and without a
gate/rain-station selected (so `.gis-side-column` is present), and confirm (a) the map canvas fills
available height rather than looking squeezed, and (b) the detail-card is fully visible and
scrollable rather than clipped.

## 9. Gotchas

- If `.gis-page-grid` ever gains a second genuinely flexible row (e.g. a bottom row reintroduced,
  mirroring `.gis-bottom-row`'s pre-2026-08-11 removal noted in the base-rule comment), the
  single-flexible-track assumption behind this fix breaks and the `.mon-page-grid`-style clamp
  pattern would become relevant again — don't blindly re-apply `minmax(.., 1fr)` without re-checking
  track count first.
- The 360px/280px floors are guesses at "won't visually collapse," not measured minimums for
  `.gis-side-column`'s actual content (detail panel + camera card). If a future session adds more
  content to that column, the floor may need raising.
- This fix was not visually verified — treat it as a plausible geometry fix pending real-browser
  confirmation, not a closed bug.
