# Session 2026-08-28 — Comparison page decision rail (Phase 5)

Project: DSS_Song_Ray · Branch: `worktree-comparison-page-redesign` · Commit: `95bd675`

## 1. Requirement recap

Phase 5 of the comparison-page-professional-redesign plan asked to replace the "decision rail"
sidebar on the comparison page. The old panel always showed a green "Khuyến nghị" (Recommended)
badge with a trophy icon no matter what the backend actually computed — misleading in a flood
operations tool where a scenario might be `priority-review`, `needs-expert`, or outright
`not-recommendable`. The panel also embedded a 180×100 SVG semicircular gauge just to show one
confidence number, eating ~130px of a sticky rail.

The backend side of this feature (an earlier same-day session, see the `backend-api` category
report `SESSION-2026-08-28-DSS_Song_Ray-comparison-backend-recommendation-engine.md`) already
exposed `ComparisonRecommendation` with a 4-state `status`, a `confidenceBand`
(high/medium/low), a `ranking` array, and per-scenario `assessments` (strengths/violations).
This session's job was purely to make the frontend honestly reflect that data.

## 2. How it was implemented + docs used

No external docs were needed — this was a rewrite of two existing presentational components
against an already-defined backend type (`ComparisonRecommendation` in
`apps/web/src/services/comparisonService.ts`). The plan document
(`plans/260828-0913-comparison-page-professional-redesign/phase-05-decision-rail.md`) was the
spec of record.

Key decisions:
- **Edit in place, don't rename.** Both `DSSRecommendationPanel.tsx` and
  `ForecastConfidenceCard.tsx` kept their PascalCase filenames even though the rest of the
  `comparison/` folder uses kebab-case, per the plan's explicit constraint to avoid import churn
  across the page.
- **Zero client-side recomputation.** All wording, tone, and the confidence band come straight
  from `recommendation.status` / `recommendation.confidenceBand`. This directly removes the
  `buildReasons()` heuristic and the `confidenceBand()` percent-threshold function that
  previously duplicated backend logic on the frontend — those were the actual bug (always-green
  badge) since they derived a Vietnamese label from percentages/hardcoded assumptions instead of
  trusting the backend's decision.
- **`role="meter"` over a generic progress bar.** A meter represents a scalar within a known
  range (matches a confidence score); a progressbar semantically implies task completion, which
  this isn't.
- **Reused existing design tokens.** The 4 new tone classes
  (`comparison-dss-state--{success,warn,info,danger}`) reuse `--success-bg/-fg`, `--warn-bg/-fg`,
  etc. already defined for `scenario-status-badge.tsx` rather than inventing new colors — DRY.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/web/src/components/comparison/DSSRecommendationPanel.tsx` | Decision rail's main panel | Always rendered a green "Khuyến nghị" trophy badge; built its own reasons list via `buildReasons()` heuristic; took `recommendedScenario` (a `SimulationScenarioSummary`) as prop | Renders one of 4 states via `STATUS_META`; shows ranking chips, compliance count, strengths, violations, and a state-appropriate primary action button; takes `scenarioCodeById`/`scenarioNameById` maps instead of a resolved scenario object |
| `apps/web/src/components/comparison/ForecastConfidenceCard.tsx` | Confidence indicator | 180×100 SVG semicircular gauge; computed its own `confidenceBand()` from percent thresholds | Compact horizontal `role="meter"` bar; band comes from backend `confidenceBand` prop, no client threshold logic |
| `apps/web/src/pages/ComparisonPage.css` | Styles for the rail | `.comparison-dss-headline`/`-trophy`, `.comparison-confidence-gauge/-value/-label` (SVG rules) | `.comparison-dss-state--{tone}`, `.comparison-dss-ranking`, `.comparison-dss-violations` (line-clamped), `.comparison-confidence-meter/-track/-fill` (bar rules) |
| `apps/web/src/pages/ComparisonPage.tsx` | Page composing the rail | Passed a single resolved `recommendedScenario` object to the panel | Builds `scenarioCodeById`/`scenarioNameById` `Map`s via `useMemo` from `allScenarios`, passes those plus the raw `recommendation` |

## 4. Code changes in detail

### 1. Status-driven headline replaces the hardcoded trophy badge — `DSSRecommendationPanel.tsx`

**Before:**
```tsx
function buildReasons(recommendation: ComparisonRecommendation): DssReason[] {
  const preferred = recommendation.assessments.find((a) => a.scenarioId === recommendation.preferredScenarioId);
  const reasons: DssReason[] = [{ text: recommendation.summary, isWarning: recommendation.status !== "recommended" }];
  for (const strength of preferred?.strengths ?? []) {
    reasons.push({ text: strength });
  }
  return reasons;
}
...
<span className="comparison-dss-trophy" aria-hidden>
  <TrophyIcon />
</span>
<div>
  <p className="t-h3">{recommendedScenario.name}</p>
  <span className="comparison-badge comparison-badge--recommended">Khuyến nghị</span>
</div>
```

**After:**
```tsx
const STATUS_META: Record<RecommendationStatus, { label: string; tone: Tone; Icon: () => JSX.Element }> = {
  recommended: { label: "Khuyến nghị", tone: "success", Icon: CheckIcon },
  "priority-review": { label: "Ưu tiên xem xét", tone: "warn", Icon: WarningIcon },
  "needs-expert": { label: "Cần chuyên gia xác nhận", tone: "info", Icon: InfoIcon },
  "not-recommendable": { label: "Chưa thể khuyến nghị", tone: "danger", Icon: WarningIcon },
};
...
<div className={`comparison-dss-state comparison-dss-state--${meta.tone}`}>
  <span className="comparison-dss-state-icon" aria-hidden>
    <meta.Icon />
  </span>
  <div>
    <p className="t-h3">{meta.label}</p>
    {preferredScenarioId && (
      <p className="t-caption">{formatScenario(preferredScenarioId, scenarioCodeById, scenarioNameById)}</p>
    )}
  </div>
</div>
```

**What changed:** Removed `buildReasons()` (client-side reason synthesis) and `TrophyIcon`.
Added `STATUS_META`, a lookup table that is the single place the 4 backend statuses map to
Vietnamese label + tone + icon. The headline now renders `meta.label` (whichever of the 4
states applies) instead of a static "Khuyến nghị" string, and the scenario name/code line is
resolved through new `scenarioCodeById`/`scenarioNameById` maps instead of a pre-resolved
`recommendedScenario` object.

**Why:** This is the actual bug fix — the old code hardcoded a green success badge regardless
of `recommendation.status`, so a flood scenario that violated operating rules could still show
"Khuyến nghị" to an operator.

**How it behaves now:** The badge, its color, and its icon now change per scenario outcome —
e.g. a `not-recommendable` result shows a red/warning-toned "Chưa thể khuyến nghị" block instead
of a green trophy.

### 2. Ranking chips, compliance count, and violations list — new UI surfaces

**Before:** (did not exist — old panel only showed a single reasons list mixing strengths and
one warning line from `buildReasons()`)

**After:**
```tsx
{ranking.length > 0 && (
  <ol className="comparison-dss-ranking">
    {ranking.map((id) => (
      <li key={id} className={id === preferredScenarioId ? "is-preferred" : undefined}>
        {scenarioCodeById.get(id) ?? id}
      </li>
    ))}
  </ol>
)}

<p className="t-caption comparison-dss-compliance">
  {compliantCount}/{assessments.length} phương án đạt quy trình vận hành
</p>
...
{preferred && preferred.violations.length > 0 && (
  <ul className="comparison-dss-violations">
    {preferred.violations.map((text) => (
      <li key={text} title={text}>
        <span className="comparison-dss-check" aria-hidden><WarningIcon /></span>
        {text}
      </li>
    ))}
  </ul>
)}
```

**What changed:** Added an ordered ranking chip list (highlights the preferred scenario),
a compliance summary line derived by counting assessments with zero violations
(`assessments.filter((a) => a.violations.length === 0).length`), and a dedicated violations list
(danger-toned, 3-line clamp with a `title` attribute for full text on hover/overflow).

**Why:** The backend now computes ranking and per-scenario violations, but the old panel threw
all of that away and only surfaced a single flattened reasons array. Surfacing ranking and
violations separately from strengths gives the operator the actual decision inputs instead of a
pre-digested verdict.

**How it behaves now:** Even a `recommended` result now visibly shows any violations that exist
on the preferred scenario (e.g. from a non-preferred candidate's failed checks reflected
elsewhere), rather than hiding everything except a green badge.

### 3. State-aware primary action instead of two hardcoded buttons — `DSSRecommendationPanel.tsx`

**Before:**
```tsx
<button type="button" className="comparison-button comparison-button--primary">
  Xem chi tiết phương án <ArrowRightIcon />
</button>
<button
  type="button"
  className="comparison-button comparison-button--secondary"
  onClick={() => navigate("/simulation", { state: { scenarioIds: selectedIds } })}
>
  <PlayIcon /> Chạy lại mô phỏng
</button>
```

**After:**
```tsx
function handlePrimaryAction() {
  const state: { scenarioIds: string[]; focusScenarioId?: string } = { scenarioIds: selectedIds };
  if (primaryAction === "adjust-scenario" && preferredScenarioId) state.focusScenarioId = preferredScenarioId;
  navigate("/simulation", { state });
}

const actionLabel =
  primaryAction === "adjust-scenario" && preferredScenarioId
    ? `Điều chỉnh ${scenarioCodeById.get(preferredScenarioId) ?? preferredScenarioId}`
    : primaryAction === "view-assessment"
      ? "Xem chi tiết đánh giá"
      : "Chạy lại mô phỏng";
...
<button type="button" className="comparison-button comparison-button--primary" onClick={handlePrimaryAction}>
  {actionLabel} <ArrowRightIcon />
</button>
```

**What changed:** Collapsed two static buttons (one with no `onClick` at all — "Xem chi tiết
phương án" was previously dead) into a single button whose label and navigation target depend on
`recommendation.primaryAction` from the backend (`adjust-scenario` / `view-assessment` /
`rerun-simulation`).

**Why:** The first button had no handler — clicking "Xem chi tiết phương án" did nothing. Rather
than wire that dead button up in isolation, the fix follows the backend's `primaryAction` field
so the call-to-action actually matches the recommendation state (e.g. "adjust" only makes sense
when the state calls for it).

**How it behaves now:** One action button, always wired, whose text and router state
(`{scenarioIds, focusScenarioId?}`) change with the recommendation's computed `primaryAction`.

### 4. SVG gauge replaced with a compact `role="meter"` bar — `ForecastConfidenceCard.tsx`

**Before:**
```tsx
function confidenceBand(percent: number): { label: string; color: string } {
  if (percent >= 70) return { label: "Cao", color: "#16a34a" };
  if (percent >= 40) return { label: "Trung bình", color: "#d97706" };
  return { label: "Thấp", color: "#dc2626" };
}

export function ForecastConfidenceCard({ percent }: { percent: number }) {
  const clamped = Math.max(0, Math.min(100, percent));
  const band = confidenceBand(clamped);
  const dashOffset = CIRCUMFERENCE_HALF * (1 - clamped / 100);
  return (
    <div className="card comparison-confidence-card">
      <h4 className="comparison-confidence-title">Độ tin cậy dự báo</h4>
      <svg viewBox="0 0 180 100" className="comparison-confidence-gauge" role="img" ...>
        ... two <path> arcs with strokeDasharray/strokeDashoffset ...
        <text x="90" y="80" textAnchor="middle" className="comparison-confidence-value">{clamped}%</text>
      </svg>
      <p className="comparison-confidence-label" style={{ color: band.color }}>{band.label}</p>
      <p className="t-caption comparison-confidence-scale">Thang đo 0-100%</p>
    </div>
  );
}
```

**After:**
```tsx
const BAND_META: Record<ConfidenceBand, { label: string; tone: "success" | "warn" | "danger" }> = {
  high: { label: "Cao", tone: "success" },
  medium: { label: "Trung bình", tone: "warn" },
  low: { label: "Thấp", tone: "danger" },
};

export function ForecastConfidenceCard({ percent, band }: { percent: number; band: ConfidenceBand }) {
  const clamped = Math.max(0, Math.min(100, percent));
  const meta = BAND_META[band];
  return (
    <div className="comparison-confidence-meter">
      <div className="comparison-confidence-meter-head">
        <span className="t-label comparison-confidence-title">Độ tin cậy dự báo</span>
        <span className={`comparison-confidence-value comparison-confidence-value--${meta.tone}`}>
          {clamped}% · {meta.label}
        </span>
      </div>
      <div
        className="comparison-confidence-track"
        role="meter"
        aria-valuenow={clamped}
        aria-valuemin={0}
        aria-valuemax={100}
        aria-label={`Độ tin cậy dự báo, mức ${meta.label}`}
      >
        <div className={`comparison-confidence-fill comparison-confidence-fill--${meta.tone}`} style={{ width: `${clamped}%` }} />
      </div>
      <p className="t-caption comparison-confidence-scale">Thang đo 0-100%</p>
    </div>
  );
}
```

**What changed:** Removed the SVG arc-drawing math (`RADIUS`, `STROKE`, `CIRCUMFERENCE_HALF`,
`strokeDasharray`/`strokeDashoffset`) and the client-side `confidenceBand()` percent-threshold
function. Replaced with a `band` prop consumed directly from the backend, and a plain `<div>`
bar sized via inline `width` percent, semantically tagged `role="meter"` with ARIA value
attributes.

**Why:** Two problems at once — the gauge consumed ~130px of vertical space in a sticky sidebar
for a single number, and `confidenceBand()` duplicated decision logic that the backend now owns
(`ComparisonRecommendation.confidenceBand`), risking drift between what the badge says and what
the gauge says if the two thresholds ever diverged.

**How it behaves now:** A compact ~30px bar; band label/tone is guaranteed consistent with the
main recommendation state because both read from the same backend field.

### 5. Page wiring — scenario code/name lookup maps — `ComparisonPage.tsx`

**Before:**
```tsx
const recommendedScenarioId = result?.recommendation?.preferredScenarioId ?? null;
const recommendedScenario = allScenarios.find((s) => s.id === recommendedScenarioId) ?? null;
...
<DSSRecommendationPanel
  recommendedScenario={recommendedScenario}
  recommendation={result?.recommendation ?? null}
  selectedIds={selectedIds}
/>
```

**After:**
```tsx
const scenarioCodeById = useMemo(
  () => new Map(allScenarios.map((s, index) => [s.id, scenarioCode(index)])),
  [allScenarios],
);
const scenarioNameById = useMemo(() => new Map(allScenarios.map((s) => [s.id, s.name])), [allScenarios]);
...
<DSSRecommendationPanel
  recommendation={result?.recommendation ?? null}
  scenarioCodeById={scenarioCodeById}
  scenarioNameById={scenarioNameById}
  selectedIds={selectedIds}
/>
```

**What changed:** Replaced the single resolved `recommendedScenario` object with two `Map`s
built via `useMemo`, using the existing `scenarioCode(index)` helper from `lib/comparison-mock.ts`
(the same PA1..PA4 identity codes used elsewhere on the page).

**Why:** The rewritten panel needs to resolve *any* scenario id referenced in
`ranking`/`assessments` (not just the single preferred one) to a short code and full name, for
the ranking chips and violation attribution.

**How it behaves now:** Ranking chips and the preferred-scenario line both render consistent
PA1..PA4-style codes matching the rest of the comparison page, sourced from one shared derivation
instead of duplicating scenario-lookup logic per component.

## 5. How to find this again

- Component: `apps/web/src/components/comparison/DSSRecommendationPanel.tsx`
- Component: `apps/web/src/components/comparison/ForecastConfidenceCard.tsx`
- Lookup table: `STATUS_META` (4-state status → label/tone/icon), `BAND_META` (confidence band → label/tone)
- Backend type: `ComparisonRecommendation`, `RecommendationStatus` in `apps/web/src/services/comparisonService.ts`
- CSS: `.comparison-dss-state--`, `.comparison-dss-ranking`, `.comparison-dss-violations`,
  `.comparison-confidence-meter` in `apps/web/src/pages/ComparisonPage.css`
- Plan: `plans/260828-0913-comparison-page-professional-redesign/phase-05-decision-rail.md`

## 6. Concepts introduced

- **ARIA `role="meter"`**: represents a scalar measurement within a known range (distinct from
  `role="progressbar"`, which implies task completion progress). Used here because a confidence
  score is a static value, not an in-progress operation — needed for the new bar to be
  accessible to screen readers with correct semantics.
- No other new concepts; the rest of the session was a rewrite of existing patterns (lookup
  tables, `useMemo`-derived maps) already used elsewhere in this codebase.

## 7. Where it got stuck

No compile errors or runtime bugs were hit this session — `npx tsc --noEmit` passed clean on the
first pass after the full rewrite of both components. There was one deliberate scope boundary
worth recording since it could look like an oversight to a future reader rather than a decision:

- **Observation:** `DSSRecommendationPanel`'s primary action button now navigates to
  `/simulation` with router state `{ scenarioIds, focusScenarioId? }` for the `adjust-scenario`
  case, but the `/simulation` page/route does not currently read or act on that
  `focusScenarioId` state.
- **Decision (not a bug):** left unimplemented on purpose. The phase-05 plan scoped this session
  to the comparison-page rail only; wiring `/simulation` to consume the router state is a
  separate downstream concern (a prior phase's report already flagged the same gap for the
  `scenarioIds` field). Recorded here explicitly so it isn't rediscovered as a mystery later —
  the state is being *sent* correctly, it just isn't *consumed* yet on the receiving page.

## 8. Verify

```
cd apps/web
npx tsc --noEmit -p .
```
Passed clean with no output (no type errors). No automated test suite covers this frontend
component pair; verification was compile-only for this presentational rewrite.

## 9. Gotchas

- If a 5th `RecommendationStatus` value is ever added on the backend, `STATUS_META` is a
  `Record<RecommendationStatus, ...>` so TypeScript will fail to compile until the new case is
  added here — that's intentional (exhaustiveness), don't silence it with a default case.
- `ForecastConfidenceCard`'s `band` prop must stay in sync with the backend's
  `ComparisonRecommendation.confidenceBand` type (`"high" | "medium" | "low"`); if that union
  changes, `BAND_META` needs a matching update or it will fail to compile (same exhaustiveness
  guard).
- The violations list is 3-line clamped via `-webkit-line-clamp` (works in all supported
  browsers here since it's Chromium-based Electron/browser targets); if this codebase ever needs
  Firefox-specific support, the clamp CSS would need testing there too.
- `scenarioCodeById`/`scenarioNameById` are derived from `allScenarios` array *index*, so if the
  scenario list is ever reordered independently of the backend's stable `scenarioCode()` mapping,
  codes could shift — verify `scenarioCode(index)` in `comparison-mock.ts` still ties codes to a
  stable identity, not array position, if scenarios become dynamically reorderable later.
- `/simulation` route/page does not yet consume `{ scenarioIds, focusScenarioId }` router state —
  see section 7. Don't assume clicking "Điều chỉnh PAx" actually focuses that scenario yet.
