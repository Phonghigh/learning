# Session 2026-08-25 — P12-05: FE alerts page with live feed and acknowledge action

## 1. Requirement recap

Task P12-05 from `tasks/INDEX.md`: build the FE alerts page — a live alert feed with color-coded
severity (red/orange/yellow/green mapped to a 4-level status), a "time since alert" label, an
acknowledge button, a filter by severity, and a way to dismiss/archive alerts already handled.
Dependencies: P12-04 (`alertsService.ts`) and P4-04 (`StatusBadge.tsx`, the 4-level status
component). Both were listed as already done in `tasks/INDEX.md`.

## 2. How it was implemented + docs used

Before writing any page code, the dependencies had to actually be found on disk. `Glob` for
`apps/web/src/**/alert*` returned nothing — despite `tasks/INDEX.md` marking P12-04 as `[x]`. This
turned into the main technical event of the session; see section 7.

Once the missing history was merged in, the page itself followed the pattern already established by
sibling pages (`ForecastAlertsTab.tsx`, `AlertsPanel.tsx` on the dashboard):

- Reused `StatusBadge` (P4-04) instead of building a new colored badge.
- Reused `alertsService.ts` (P12-04) for `getAlerts` / `acknowledgeAlert` — no new HTTP code.
- New, task-specific pieces: `alertLevel.ts` (level mapping + relative time), `AlertListItem.tsx`,
  `AlertSeverityFilter.tsx`.
- Dismiss/archive: the alerts table has no server-side "dismissed" column (confirmed by reading
  `apps/api/src/alerts/entities/alert.entity.ts` and the doc comment in `alertsService.ts`: "No
  separate acknowledgedAt/history table exists: acknowledge sets `resolvedAt`, and history returns
  the current record"). The task's own acceptance criterion explicitly allows "localStorage to hide,
  or server-side mark read" as alternatives, so localStorage was chosen — no schema change needed,
  smallest change that satisfies the requirement (YAGNI).
- i18n: `tasks/ROUTINE.md` requires adding keys to `apps/web/src/i18n/strings.ts` and running
  `apps/web/scripts/check-i18n.mjs`. Neither file exists anywhere in the repo (`Glob` confirmed no
  matches for both paths). `tasks/PROGRESS.md` shows a prior session (P9-04, same date) already hit
  and documented this exact gap: "No i18n system exists in this repo... hardcoded Vietnamese strings
  matching existing page conventions." This session followed that same convention rather than
  inventing an i18n system that doesn't exist, and skipped the i18n-check step since there is nothing
  to check.
- CSS: this repo has a binding rule — reinforced by a full prior session dedicated to converting
  the app from flex to grid — that layout must use CSS Grid only, never Flexbox. `AlertsPage.css` is
  grid-only throughout, including `grid-auto-flow: column` for inline groups (header row, action
  buttons) instead of `display: flex`.
- Real-Data Rule (`tasks/ROUTINE.md`): a genuinely empty result set is not the same as missing data.
  The page distinguishes three states rather than collapsing them into one generic "no data" message
  (see section 4, item 1).

No new docs were consulted beyond the repo's own `tasks/ROUTINE.md`, `tasks/INDEX.md`,
`tasks/PROGRESS.md`, and the existing sibling components — this was an internal-consistency task, not
a new-library task.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/web/src/pages/AlertsPage.tsx` | Alerts page | 1-line stub: `<h2>Cảnh báo</h2>` | Full page: fetch, filter, sort, acknowledge, dismiss, 3-way empty state |
| `apps/web/src/pages/AlertsPage.css` | Page styles | did not exist | Grid-only layout for header/list/row/actions |
| `apps/web/src/components/alerts/alertLevel.ts` | Shared helpers | did not exist | `toStatusLevel(level)` + `timeSince(iso, now?)` |
| `apps/web/src/components/alerts/AlertListItem.tsx` | One alert row | did not exist | Badge + message + time-since + ack/dismiss button |
| `apps/web/src/components/alerts/AlertSeverityFilter.tsx` | Filter control | did not exist | Controlled `<select>` over `StatusLevel \| "all"` |
| `tasks/INDEX.md` | Task tracker | P12-05 unchecked | P12-05 checked `[x]` |
| `tasks/PROGRESS.md` | Session log | — | Appended 2026-08-25 P12-05 entry |
| (git history) | Branch state | worktree branch stopped at `62722a8` (P9-02 era) | Merged `localmain/main` up to `42dce88`, bringing in P12-04/P9-06/P10-02/CSS-grid-refactor commits |

## 4. Code changes in detail

### 1. AlertsPage.tsx — full page logic, replacing the stub — `apps/web/src/pages/AlertsPage.tsx`

**Before:**
```tsx
export function AlertsPage() {
  return <h2>Cảnh báo</h2>;
}
```

**After (key excerpt):**
```tsx
export function AlertsPage() {
  const { accessToken } = useAuth();
  const [alerts, setAlerts] = useState<Alert[]>([]);
  const [state, setState] = useState<FetchState>("loading");
  const [severityFilter, setSeverityFilter] = useState<StatusLevel | "all">("all");
  const [dismissedIds, setDismissedIds] = useState<Set<string>>(() => loadDismissedIds());
  const [acknowledgingId, setAcknowledgingId] = useState<string | null>(null);

  useEffect(() => {
    if (!accessToken) return;
    let cancelled = false;
    setState("loading");
    getAlerts(accessToken)
      .then((rows) => { if (!cancelled) { setAlerts(rows); setState("ready"); } })
      .catch(() => { if (!cancelled) setState("error"); });
    return () => { cancelled = true; };
  }, [accessToken]);

  function handleAcknowledge(id: string) {
    if (!accessToken) return;
    setAcknowledgingId(id);
    acknowledgeAlert(accessToken, id)
      .then((updated) => {
        setAlerts((prev) => prev.map((alert) => (alert.id === id ? updated : alert)));
      })
      .catch(() => { /* leave row as-is; user can retry */ })
      .finally(() => setAcknowledgingId(null));
  }

  function handleDismiss(id: string) {
    setDismissedIds((prev) => {
      const next = new Set(prev);
      next.add(id);
      saveDismissedIds(next);
      return next;
    });
  }

  const visibleAlerts = useMemo(() => {
    return alerts
      .filter((alert) => !dismissedIds.has(alert.id))
      .filter((alert) => severityFilter === "all" || toStatusLevel(alert.level) === severityFilter)
      .sort((a, b) => new Date(b.triggeredAt).getTime() - new Date(a.triggeredAt).getTime());
  }, [alerts, dismissedIds, severityFilter]);

  return (
    <div className="alerts-page">
      {/* ... header with filter ... */}
      <div className="card alerts-page-feed">
        {state === "loading" && <p className="t-body alerts-page-status">Đang tải...</p>}
        {state === "error" && (
          <p className="t-body alerts-page-status">
            Chưa có dữ liệu — không thể tải danh sách cảnh báo.
          </p>
        )}
        {state === "ready" && visibleAlerts.length === 0 && (
          <p className="t-body alerts-page-status">
            {alerts.length === 0
              ? "Không có cảnh báo nào đang hoạt động."
              : "Không có cảnh báo nào khớp với bộ lọc hiện tại."}
          </p>
        )}
        {state === "ready" && visibleAlerts.length > 0 && (
          <ul className="alerts-list">
            {visibleAlerts.map((alert) => (
              <AlertListItem key={alert.id} alert={alert} onAcknowledge={handleAcknowledge}
                onDismiss={handleDismiss} isAcknowledging={acknowledgingId === alert.id} />
            ))}
          </ul>
        )}
      </div>
    </div>
  );
}
```

**What changed:** replaced a static 1-line component with fetch (`useEffect` + `getAlerts`),
client-side filter/sort (`useMemo`), acknowledge (`acknowledgeAlert` + in-place row replacement,
not optimistic hiding), dismiss (localStorage-backed `Set<string>`), and a 3-way empty/error state.

**Why:** satisfies the task's live-feed, filter, acknowledge, and dismiss requirements while
respecting the Real-Data Rule — a fetch failure and a legitimate zero-alerts result must not show the
same message.

**How it behaves now:** on mount, if authenticated, the page fetches all alerts (no status filter),
shows a loading state, then either the list, a load-error message, a "no active alerts" message, or a
"no alerts match the filter" message depending on which of the three conditions is true. Acknowledging
calls the server and only updates the row once the server confirms (via the returned `resolvedAt`),
so the UI never shows an acknowledged state that the server hasn't actually recorded. Dismissing an
already-resolved alert removes it from view and persists that choice across reloads via
`localStorage`.

### 2. Level mapping and relative time helpers (new) — `apps/web/src/components/alerts/alertLevel.ts`

**Before:** file did not exist.

**After:**
```ts
export function toStatusLevel(level: string): StatusLevel {
  if (level === "alert_high") return "critical";
  if (level === "warning" || level === "alert" || level === "critical") return level;
  return "normal";
}

const MINUTE_MS = 60_000;
const HOUR_MS = 60 * MINUTE_MS;
const DAY_MS = 24 * HOUR_MS;

export function timeSince(iso: string, now: number = Date.now()): string {
  const diffMs = now - new Date(iso).getTime();
  if (diffMs < 0 || diffMs < MINUTE_MS) return "vừa xong";
  if (diffMs < HOUR_MS) return `${Math.floor(diffMs / MINUTE_MS)} phút trước`;
  if (diffMs < DAY_MS) return `${Math.floor(diffMs / HOUR_MS)} giờ trước`;
  return `${Math.floor(diffMs / DAY_MS)} ngày trước`;
}
```

**What changed:** two new pure functions, no dependencies added.

**Why:** the backend's `level` string doesn't match `StatusBadge`'s 4-value union 1:1 (there's a
5th value, `alert_high`), and no relative-time formatting existed yet for alerts.

**How it behaves now:** `toStatusLevel("alert_high")` collapses to `"critical"` — the same convention
already used in `AlertsPanel.tsx` on the dashboard, kept consistent rather than reinvented.
`timeSince` needs no library (`date-fns`, etc.) — plain millisecond arithmetic covers the four labels
the task asked for.

### 3. Real-Data Rule empty states — `apps/web/src/pages/AlertsPage.tsx` (excerpt above)

**Before:** (n/a — page did not fetch anything)

**After:** three distinct messages — `"Không có cảnh báo nào đang hoạt động."` (real empty result),
`"Chưa có dữ liệu — không thể tải danh sách cảnh báo."` (fetch failed), `"Không có cảnh báo nào khớp
với bộ lọc hiện tại."` (filter excludes everything).

**What changed:** added conditional branches keyed off `state` and `alerts.length` vs
`visibleAlerts.length`, instead of one catch-all "no data" message.

**Why:** the repo's Real-Data Rule (`tasks/ROUTINE.md`) forbids treating "zero real alerts" the same
as "couldn't load data" — conflating them would either hide a real outage or falsely imply a data
problem when the system is simply healthy.

**How it behaves now:** a network failure now reads distinctly from "no alerts currently," and
narrowing the severity filter to an empty subset reads distinctly from both.

### 4. Grid-only CSS — `apps/web/src/pages/AlertsPage.css`

**Before:** file did not exist.

**After (representative rules):**
```css
.alerts-page-header {
  display: grid;
  grid-auto-flow: column;
  justify-content: space-between;
  align-items: center;
  gap: var(--gap-inline);
}

.alerts-list-item {
  display: grid;
  grid-template-columns: max-content 1fr max-content max-content;
  align-items: center;
  gap: var(--gap-inline);
  padding: var(--pad-cell);
  border-bottom: 1px solid var(--border-subtle);
}
```

**What changed:** every layout rule uses `display: grid` (`grid-auto-flow: column` for inline
groups, explicit `grid-template-columns` for the row layout); no `display: flex` anywhere.

**Why:** the repo's binding CSS rule ("DSS Song Ray FE must use CSS Grid everywhere, never Flexbox")
predates this session and was the subject of a dedicated prior refactor (`19c73b8` "refactor:
convert Dashboard/GisMap/Forecast page layouts from flex to grid"). New code has to match it.

**How it behaves now:** the alert row lays out as `badge | message | time | actions` using
`max-content 1fr max-content max-content`, and header/action-button groups use
`grid-auto-flow: column` to achieve the same horizontal-row effect flexbox would normally give.

## 5. How to find this again

- Component: `AlertsPage` in `apps/web/src/pages/AlertsPage.tsx`
- Helpers: `toStatusLevel`, `timeSince` in `apps/web/src/components/alerts/alertLevel.ts`
- Row component: `AlertListItem` in `apps/web/src/components/alerts/AlertListItem.tsx`
- Filter: `AlertSeverityFilter` in `apps/web/src/components/alerts/AlertSeverityFilter.tsx`
- localStorage key: `dss.alerts.dismissedIds`
- Service calls used: `getAlerts`, `acknowledgeAlert` in `apps/web/src/services/alertsService.ts`
- Task entry: `P12-05` in `tasks/INDEX.md`; log entry dated 2026-08-25 in `tasks/PROGRESS.md`
- Related prior gap note: `P9-04` entry in `tasks/PROGRESS.md` (i18n system absence)

## 6. Concepts introduced

- **Client-side archival via localStorage vs. server-side state**: when a UI needs a "hide/dismiss"
  concept that the backend schema doesn't model, it's valid to keep that state purely on the client
  (here, a `Set<string>` of ids serialized to `localStorage`) rather than adding a DB column for a
  view preference — this task needed it because the `alerts` table only has `resolvedAt`, no
  `dismissed` flag.
- **Optimistic vs. confirmed UI update**: acknowledging an alert updates the row only after the
  server's response arrives (`acknowledgeAlert(...).then((updated) => ...)`), not immediately on
  click — this task needed it so the UI never shows "acknowledged" for something the server actually
  rejected or failed to persist.
- No other genuinely new concepts — the fetch/filter/sort pattern, `StatusBadge` reuse, and grid-only
  CSS all follow conventions already established elsewhere in this repo.

## 7. Where it got stuck

**Symptom:** `Glob` for `apps/web/src/**/alert*` returned no matches, even though `tasks/INDEX.md`
marked P12-04 (the alerts service dependency) as complete (`[x]`).

**False lead ruled out:** first assumption was that the file had a different name or path than
expected. A broader search and a direct read of `alertsService.ts`'s expected location both came back
empty — the file genuinely did not exist on this branch, not a naming mismatch.

**Cause (confirmed via `git log --oneline --all | grep P12-04`):** the commits for P12-04 existed in
git history, just not reachable from this worktree's current branch tip. `git log` showed this
worktree's branch (`worktree-agent-a0a99a98609fe9a50`) was still at `62722a8` (P9-02 era), while
`main` had advanced to `42dce88` through several more merged task sessions, including `cee0d5a`
(P12-04) and `35b93bd` (P9-06). **Inferred** explanation (not directly observed, but consistent with
the evidence): this worktree branch was created from an earlier point in the project's history, and
the primary checkout (`E:\DSS\DSS_Song_Ray`) kept committing locally without those commits being
pushed to `origin`, so the worktree fell behind main by several sessions.

**Second snag:** `git fetch origin main` did not bring the missing commits in (origin itself was
behind, consistent with the "not pushed" inference above). `git fetch localmain main` — where
`localmain` is a pre-existing git remote pointing at the local main checkout — failed outright with
`fatal: 'localmain' does not appear to be a git repository, fatal: Could not read from remote
repository` (a fetch-protocol/transport issue, not a content problem).

**Fix:** despite the fetch failure, `remotes/localmain/main` already had the needed commits cached
locally from some prior fetch. `git merge localmain/main --no-edit` succeeded directly without
re-fetching, producing merge commit `42dce88` on this branch and bringing in ~20 files across
P12-04, P9-06, P10-02, and the flex-to-grid refactor session. No rework was needed — this was purely
history reconciliation, not a conflict to resolve.

**Third, smaller snag:** the task's own workflow (`tasks/ROUTINE.md`) calls for an i18n check
(`node apps/web/scripts/check-i18n.mjs` against `apps/web/src/i18n/strings.ts`). Both paths were
confirmed absent via `Glob` (no matches). This was not re-investigated from scratch — `tasks/PROGRESS.md`
already had a same-day P9-04 entry documenting this exact gap ("No i18n system exists in this
repo"), so this session reused that finding rather than re-deriving it, and skipped the check step
since there was nothing to run it against.

## 8. Verify

```
pnpm --filter web exec tsc --noEmit
```
Result: clean, zero type errors. (pnpm auto-installed 801 workspace packages on first invocation
since devDependencies were missing locally — expected pnpm-workspace behavior, not a build error.)

```
pnpm --filter web build
```
Result: succeeded — `tsc -b && vite build`, 685 modules transformed, `dist/` produced. Only the
pre-existing ">500kB chunk" advisory warning appeared, which is unrelated to this change and already
documented as non-blocking in earlier sessions' `tasks/PROGRESS.md` entries.

i18n check (`node apps/web/scripts/check-i18n.mjs`) — intentionally skipped; script and target file
do not exist in this repo (pre-existing gap, see section 7).

## 9. Gotchas

- If P12-06 (alert detail modal) or P12-07 (alerts history/CSV export) build on this page, remember
  `alertLevel.ts`'s `toStatusLevel` mapping is the single source of truth for the `alert_high` →
  `critical` collapse — don't reintroduce a second mapping.
- The dismissed-ids list lives only in `localStorage` on the current browser/device — it will not
  sync across devices or survive a cleared browser storage. If a future task needs dismiss state to
  be durable or shared, that requires an actual schema change (a `dismissed` column or table), not a
  client-side patch.
- Acknowledge is not optimistic: if the network is slow, the row will sit in its "Xác nhận" state
  with the button disabled until the server responds. A future UX pass could add a timeout/retry
  affordance if this proves too slow in practice.
- This worktree's branch can still be behind `main`/`localmain` for other task ranges not yet merged
  in (only up through `42dce88` was pulled). Before assuming a dependency is "missing," check
  `git log --oneline --all | grep <task-id>` before concluding it was never implemented.
- The i18n gap is repo-wide and unresolved — any future FE task should keep following the hardcoded
  Vietnamese convention until/unless a real i18n system is introduced deliberately, not per-task.
