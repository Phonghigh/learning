# Session 2026-09-04 — KpiCard reworked into a 3-row centered layout

## 1. Requirement recap

Follow-up to an earlier fix in the same session (already logged separately: adding
`justify-self: center` to `.kpi-card-value`). User then specified a bigger layout change directly as
CSS/pseudo-code, in Vietnamese: rework `.kpi-card` into a 3x2 grid — row 1 = title spanning both
columns, row 2 = icon (col 1) + value (col 2), row 3 = caption spanning both columns — "tất cả đều
có justify-self: center" (everything gets `justify-self: center`).

## 2. How it was implemented + docs used

No new docs needed — this is a direct application of CSS Grid `grid-template-areas` reshuffling,
which the codebase already uses everywhere per its Grid-only convention. The user's spec was
followed literally: changed the area map, added `justify-self: center` to every grid child, and (one
addition beyond the literal spec) `justify-items: center` on `.kpi-card-caption` since it is itself a
nested grid container with up to 2 lines of text — `justify-self` alone only centers the caption box
as a unit within its parent cell, not the lines inside it. No alternatives were considered; this was
a direct, unambiguous instruction.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/web/src/components/dashboard/KpiCard.css` | Dashboard KPI card layout | 2-col x 3-row grid: icon spans rows 1-2 on the left, title/value stacked on the right, caption full-width below | 2-col x 3-row grid: title spans row 1 full-width, icon+value share row 2, caption spans row 3 full-width; every child centered via `justify-self`/`justify-items` |

## 4. Code changes in detail

### 1. Grid template — columns and areas — `apps/web/src/components/dashboard/KpiCard.css`

**Before:**
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

**After:**
```css
.kpi-card {
  padding: var(--sp-2) var(--sp-3);
  display: grid;
  grid-template-columns: 38px 1fr;
  grid-template-areas:
    "title title"
    "icon value"
    "caption caption";
  align-items: center;
  column-gap: var(--sp-2);
  row-gap: 1px;
  border: 1px solid #dfe1f7;
}
```

**What changed:** `grid-template-columns` changed from `auto 1fr` (icon column sized to its own
content) to a fixed `38px 1fr`. `grid-template-areas` changed the top row from `"icon title"` (icon
spanning rows 1-2, title as a lone right-column cell) to `"title title"` (title now spans the full
width as its own row); the icon moved down into row 2 alongside value.

**Why:** This is a genuinely different composition, not a tweak — previously the icon sat beside two
stacked lines of text; now the title is a small header line above a centered icon+number row. This
was the user's own explicit spec.

**How it behaves now:** The card renders as title on top (centered, full width), icon and value
side by side in the middle row, caption at the bottom (centered, full width).

### 2. Centering every grid child — `apps/web/src/components/dashboard/KpiCard.css`

**Before:**
```css
.kpi-card-icon {
  grid-area: icon;
  align-self: center;
}

.kpi-card-title {
  grid-area: title;
  font-size: var(--fs-caption);
  ...
}

.kpi-card-caption {
  grid-area: caption;
  margin-top: var(--sp-1);
  display: grid;
  gap: 2px;
  ...
}

.kpi-card-no-data {
  grid-area: value;
  color: var(--text-5);
  font-style: italic;
}
```

**After:**
```css
.kpi-card-icon {
  grid-area: icon;
  align-self: center;
  justify-self: center;
}

.kpi-card-title {
  grid-area: title;
  justify-self: center;
  font-size: var(--fs-caption);
  ...
}

.kpi-card-caption {
  grid-area: caption;
  justify-self: center;
  margin-top: var(--sp-1);
  display: grid;
  justify-items: center;
  gap: 2px;
  ...
}

.kpi-card-no-data {
  grid-area: value;
  justify-self: center;
  color: var(--text-5);
  font-style: italic;
}
```

**What changed:** Added `justify-self: center` to `.kpi-card-icon`, `.kpi-card-title`,
`.kpi-card-caption`, `.kpi-card-no-data`. (`.kpi-card-value` already had it from the prior fix
earlier in this session.) Additionally added `justify-items: center` to `.kpi-card-caption`.

**Why:** `justify-self` follows the user's literal instruction — every grid-area child centers
itself horizontally in its own cell, matching the project's standing CSS-Grid-only convention (no
Flexbox). `.kpi-card-caption` needed the extra `justify-items: center` because it is `display: grid`
itself (it holds up to 2 lines, e.g. a water-level delta line) — `justify-self` only centers the
caption box as a whole in the parent grid; `justify-items` centers its own children (the text lines)
within itself.

**How it behaves now:** All KPI card content — icon, title, value, caption lines, and the no-data
fallback — is now horizontally centered, instead of left-aligned against the old icon column.

## 5. How to find this again

Grep for `.kpi-card` in `apps/web/src/components/dashboard/KpiCard.css`; grid areas `title`, `icon`,
`value`, `caption`; component `KpiCard.tsx` renders these class names.

## 6. Concepts introduced

- **`justify-self` vs `justify-items` on a nested grid container**: `justify-self` (set on a grid
  child) controls how *that item* positions within the single cell its parent grid assigned it.
  `justify-items` (set on a grid container) controls the default alignment of *that container's own
  children* within its own cells. `.kpi-card-caption` needed both because it plays both roles at
  once — it's a child of `.kpi-card`'s grid (needs `justify-self` to center itself) and a grid
  container of its own text lines (needs `justify-items` to center those lines).

## 7. Where it got stuck

No struggle this session. `pnpm --filter web build` (`tsc -b && vite build`) passed clean on the
first attempt after the edit — a pure CSS restructuring with no build-affecting syntax.

## 8. Verify

`pnpm --filter web build` — passes with no TypeScript or Vite errors (this is a CSS-only change, so
the check is purely that the build pipeline doesn't choke on the file; visual confirmation is via
the dashboard screenshot per the project's UI-fix rule).

## 9. Gotchas

The icon column is now a hardcoded `38px`, not tied to any `--icon-badge-size-*` design token (the
existing `--icon-badge-size-md` token elsewhere is 30px). If the icon badge component's actual
rendered size changes later, this column width will silently drift out of sync and need manual
re-alignment — it is no longer `auto`-sized to the icon's real content.
