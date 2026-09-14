# Session 2026-09-03 — Dead-code removal, SimulationPage hook extraction, OperationLogPage layout fix

## 1. Requirement recap

Three unrelated cleanup tasks in one session:

1. Remove `FloodMapPage.tsx` if it was confirmed unused (dead code discovered while searching for a `/flood` route).
2. `SimulationPage.tsx` had grown to 74 lines but was nearly unreadable — a single line contained 9 `useState` calls, and every effect/handler was crammed onto one line each, making diffs and reviews painful. Fix the readability without changing behavior.
3. While looking at the simulation page's sibling pages, `OperationLogPage.css` was found using a negative-margin/height hack to escape the shell's default page padding — worth checking whether it was still needed.

## 2. How it was implemented + docs used

**FloodMapPage removal:** grepped the whole `apps/web/src` tree for `FloodMapPage` before touching anything. The only hits were in the source file itself and in historical `plans/`/report markdown files (left untouched, since those are a record of past work, not live code). No route (`react-router-dom`) referenced it. Confirmed safe to delete outright — no route cleanup needed because it was never wired to a route in the first place.

**SimulationPage refactor:** asked the user (via `AskUserQuestion`) whether to do a minimal one-statement-per-line reformat or a full custom-hook extraction. User chose the hook extraction. Implementation split the 74-line component into:
- `use-simulation-workbench.ts` — a `useSimulationWorkbench()` hook owning all state (`lookups`, `config`, `scenarioId`, `scenarios`, `run`, `loadError`, `notice`, `scheduleOpen`, `sourceDrawer`, `activeTimestamp`, `attemptedSubmit`, `sidebarOpen`), derived values (`errors`, `status`, `isRunning`, `hasRun`), and handlers (`handleSave`, `handleRun`, `handleCancel`, `handleExport`, `handleSelectScenario`, `handleClone`, `handleConfigChange`, `handleEditSchedule`).
- `workbench-validation.ts` — `validateScenarioConfig` (renamed from the page-local `validate`), the internal `validateSource`, and `messageFor`, pulled out so the hook file stays focused on state/effects rather than validation logic.
- `SimulationPage.tsx` rewritten to call `const workbench = useSimulationWorkbench()`, destructure what JSX needs, and render — no business logic left in the component.

Verified with `pnpm build` (`tsc -b && vite build`) in `apps/web` both before and after the change — passed clean, only the pre-existing (unrelated) chunk-size warning present in both runs. Also grepped for the old inline `validate`/`messageFor` names to confirm nothing outside the moved files still referenced them.

**OperationLogPage fix:** per this project's `CLAUDE.md` FE UI/UX Fix Rule (ask before touching CSS/markup for a reported UI/UX issue), used `AskUserQuestion` to ask whether the user wanted `OperationLogPage` to keep its full-bleed layout (and standardize the flush-page pattern app-wide) or drop full-bleed for consistency with the rest of the pages. User chose to drop full-bleed. Implementation removed the now-dead `page-root--flush` mechanism entirely (`AppShell.tsx`'s `FLUSH_PAGE_PREFIXES` array + `useLocation` usage, and the CSS rule in `AppShell.css`) and let `OperationLogPage.css` fall back to the shell's default `--pad-page` padding. Verified with `pnpm build` — passed clean.

No existing code was reused for the hook/validation split beyond copy-and-relocate of the original logic; the flush-page removal deleted code rather than adding any.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/web/src/pages/FloodMapPage.tsx` | Dead page component | 3-line stub rendering `<h2>Ngập</h2>`, never routed | Deleted |
| `apps/web/src/pages/SimulationPage.tsx` | Simulation workbench page | 74 dense lines owning all state/effects/handlers inline | Thin JSX shell calling `useSimulationWorkbench()` |
| `apps/web/src/components/simulation/use-simulation-workbench.ts` | New custom hook | Did not exist | Owns all simulation state, effects, derived values, and handlers |
| `apps/web/src/components/simulation/workbench-validation.ts` | New validation module | Did not exist | Holds `validateScenarioConfig`, `validateSource`, `messageFor` |
| `apps/web/src/components/layout/AppShell.tsx` | App shell / layout wrapper | Had `FLUSH_PAGE_PREFIXES`, `useLocation`, conditional `page-root--flush` class | Always renders `className="page-root"`; no flush-page concept |
| `apps/web/src/components/layout/AppShell.css` | Shell layout styles | Had `.page-root--flush { padding: 0; }` | Rule removed |
| `apps/web/src/pages/OperationLogPage.css` | Operation log dashboard styles | `.ops-dashboard` used `margin: -20px -24px -40px; height: calc(100% + 60px); padding: var(--sp-3);` to cancel shell padding | `height: 100%;` only — relies on shell's default `--pad-page` |

## 4. Code changes in detail

### 1. Delete unused FloodMapPage component — `apps/web/src/pages/FloodMapPage.tsx`

**Before:**
```tsx
export function FloodMapPage() {
  return <h2>Ngập</h2>;
}
```

**After:**
```
(removed)
```

**What changed:** Whole file deleted (3 lines).
**Why:** Grep across `apps/web/src` confirmed zero live imports/route references — only stale mentions in historical plan/report markdown. Kept around it was pure dead code that could mislead a future search for the `/flood` path into thinking a route existed.
**How it behaves now:** The file no longer exists; nothing imported it, so there is no build or runtime change.

### 2. Extract SimulationPage state into a hook — `apps/web/src/pages/SimulationPage.tsx`

**Before (excerpt — the single dense state line and one representative handler):**
```tsx
export function SimulationPage() {
  const { accessToken } = useAuth(); const [lookups, setLookups] = useState<WorkbenchLookups | null>(null); const [config, setConfig] = useState(INITIAL_CONFIG); const [scenarioId, setScenarioId] = useState<string | null>(null); const [scenarios, setScenarios] = useState<SimulationScenarioSummary[]>([]); const [run, setRun] = useState<SimulationRun | null>(null); const [loadError, setLoadError] = useState<string | null>(null); const [notice, setNotice] = useState<string | null>(null); const [scheduleOpen, setScheduleOpen] = useState(false); const [sourceDrawer, setSourceDrawer] = useState<{ source: DataSource; title: string } | null>(null); const [activeTimestamp, setActiveTimestamp] = useState<string | null>(null); const [attemptedSubmit, setAttemptedSubmit] = useState(false); const [sidebarOpen, setSidebarOpen] = useState(false);
  useEffect(() => { if (!accessToken) return; let live = true; listScenarios(accessToken).then((rows) => live && setScenarios(rows)); return () => { live = false; }; }, [accessToken]);
  // ...several more one-line effects/handlers...
  async function handleSave() { setAttemptedSubmit(true); try { const id = await saveScenario(); setNotice(id ? "Đã lưu cấu hình kịch bản." : "Hãy hoàn tất các trường có lỗi trước khi lưu."); } catch (error) { setNotice(messageFor(error)); } }
```

**After:**
```tsx
export function SimulationPage() {
  const workbench = useSimulationWorkbench();
  const {
    lookups, config, scenarioId, scenarios, run, loadError, notice,
    scheduleOpen, sourceDrawer, activeTimestamp, attemptedSubmit, sidebarOpen,
    errors, status, isRunning, hasRun,
    setScheduleOpen, setSourceDrawer, setActiveTimestamp, setSidebarOpen,
    // ...handlers destructured here...
  } = workbench;
```
(231 new lines moved into `use-simulation-workbench.ts`, 84 into `workbench-validation.ts`; see the Remember card for what each new file owns.)

**What changed:** All `useState`/`useEffect`/`useCallback`/handler declarations were removed from the component and relocated verbatim (with the local `validate` function renamed to `validateScenarioConfig`) into the two new files. The component now only calls the hook and destructures.
**Why:** The original file mixed 9 pieces of state, 3 effects, and 7 handlers into a handful of unreadable one-liners — every diff touched the same few lines regardless of what actually changed, making code review and future edits error-prone.
**How it behaves now:** Same runtime behavior (confirmed by `pnpm build` passing both before and after with no logic rewritten, only moved) — the component is now a pure JSX-composition layer, and state/effects/handlers live in one hook that can be read, tested, or reused independently of the page markup.

### 3. Remove OperationLogPage full-bleed negative-margin hack — `apps/web/src/pages/OperationLogPage.css`

**Before:**
```css
.ops-dashboard {
  /* Cancels the light-theme page-root padding so this dashboard owns its
     own full-bleed dark surface and fits inside the shell without an
     extra scrollbar around it. */
  margin: -20px -24px -40px;
  height: calc(100% + 60px);
  display: grid;
  grid-template-rows: auto auto 1fr;
  gap: var(--sp-3);
  padding: var(--sp-3);
  background: var(--bg-app);
  color: var(--text-1);
  font-size: var(--fs-body);
```

**After:**
```css
.ops-dashboard {
  height: 100%;
  display: grid;
  grid-template-rows: auto auto 1fr;
  gap: var(--sp-3);
  background: var(--bg-app);
  color: var(--text-1);
  font-size: var(--fs-body);
```

**What changed:** Removed the `margin: -20px -24px -40px;`, the `calc(100% + 60px)` height compensation, and the redundant `padding: var(--sp-3)`; `height` is now a plain `100%`.
**Why:** The negative margin was compensating for the shell's `--pad-page` (`apps/web/src/styles.css:102`), but an equivalent `page-root--flush` mechanism already existed in `AppShell.tsx`/`AppShell.css` specifically to let a page opt out of that padding cleanly — `OperationLogPage.css` had never been updated to use it and was instead double-compensating with an older, more fragile hack.
**How it behaves now:** `.ops-dashboard` now sizes to `100%` of the shell's standard padded `page-root` container, same as every other page, instead of fighting the shell's padding with a magic-number margin.

### 4. Remove the now-unused flush-page mechanism — `apps/web/src/components/layout/AppShell.tsx` and `AppShell.css`

**Before (`AppShell.tsx`):**
```tsx
import { useState } from "react";
import { Outlet, useLocation } from "react-router-dom";
import { Sidebar } from "./Sidebar";
import { TopBar } from "./TopBar";
import "./AppShell.css";

const AUTO_COLLAPSE_BREAKPOINT = "(max-width: 1199px)";

// Pages that render their own full-bleed surface and own their spacing
// instead of the page-root's default padding — avoids the negative-margin
// height-compensation hack these pages used to need to escape that padding.
const FLUSH_PAGE_PREFIXES = ["/operation-log"];

export function AppShell() {
  const [collapsed, setCollapsed] = useState(() => window.matchMedia(AUTO_COLLAPSE_BREAKPOINT).matches);
  const { pathname } = useLocation();
  const isFlushPage = FLUSH_PAGE_PREFIXES.some((prefix) => pathname.startsWith(prefix));
  return (
    ...
        <main className={`page-root${isFlushPage ? " page-root--flush" : ""}`}>
```

**After:**
```tsx
import { useState } from "react";
import { Outlet } from "react-router-dom";
import { Sidebar } from "./Sidebar";
import { TopBar } from "./TopBar";
import "./AppShell.css";

const AUTO_COLLAPSE_BREAKPOINT = "(max-width: 1199px)";

export function AppShell() {
  const [collapsed, setCollapsed] = useState(() => window.matchMedia(AUTO_COLLAPSE_BREAKPOINT).matches);
  return (
    ...
        <main className="page-root">
```

`AppShell.css` lost the matching:
```css
/* Used by pages that render their own full-bleed surface (e.g. the
   operation-log dashboard) so they can size to exactly 100% of this
   container instead of compensating this padding with negative margins. */
.page-root--flush {
  padding: 0;
}
```
→ `(removed)`

**What changed:** Deleted `FLUSH_PAGE_PREFIXES`, the `useLocation` import and `isFlushPage` computation, and the conditional class string in `AppShell.tsx`; deleted the `.page-root--flush` rule in `AppShell.css`.
**Why:** With `OperationLogPage.css` (the only consumer of `page-root--flush`) reverted to standard padding, the flush-page mechanism had no remaining caller — keeping it would be dead code inviting future confusion about which pages need flush styling.
**How it behaves now:** `<main>` always renders `className="page-root"`; there is no longer a per-route branch in the shell's render path.

## 5. How to find this again

- `FloodMapPage` — no longer exists; historical references only in `plans/` and old report markdown.
- `useSimulationWorkbench` — `apps/web/src/components/simulation/use-simulation-workbench.ts`
- `validateScenarioConfig`, `validateSource`, `messageFor` — `apps/web/src/components/simulation/workbench-validation.ts`
- `page-root--flush`, `FLUSH_PAGE_PREFIXES` — no longer exist (grep will return nothing; useful to confirm removal if this surfaces again).
- `--pad-page` — defined in `apps/web/src/styles.css:102`, applied via `.page-root` in `apps/web/src/components/layout/AppShell.css`.
- `.ops-dashboard` — `apps/web/src/pages/OperationLogPage.css`.

## 6. Concepts introduced

- **Custom hook extraction**: pulling a React component's `useState`/`useEffect`/handler logic into a standalone `useXyz()` function so the component itself becomes a pure rendering layer. Needed here because `SimulationPage.tsx`'s inline density made review and future diffs impractical — the hook gives the state/effects a name and a location separate from JSX.
- **CSS custom property (`--pad-page`)**: a CSS variable set once (in `styles.css`) and consumed elsewhere (`AppShell.css`) to keep spacing consistent across the app without repeating literal values. Relevant here because the bug was really about a page fighting this shared variable instead of using it.
- No other new concepts — the FloodMapPage removal was a plain dead-code deletion.

## 7. Where it got stuck

- **FloodMapPage removal — false-lead check, not a snag:** before deleting, grepped for `FloodMapPage` across the whole `apps/web/src` tree specifically to rule out a hidden dynamic import or lazy-loaded route that a simple "search for `<FloodMapPage`" might miss. Found only the source file and historical markdown mentions in `plans/`/report files. This confirmed (not just assumed) that deletion was safe without any accompanying route cleanup, since no route ever pointed at it.
- **OperationLogPage — looked like a simple CSS number tweak, was actually dead-code double-compensation:** the initial read of `.ops-dashboard`'s negative margin looked like it might just need a value adjustment for some layout bug. Reading the CSS comment ("Cancels the light-theme page-root padding...") together with `AppShell.tsx`'s own comment ("...avoids the negative-margin height-compensation hack these pages used to need...") revealed that a proper `page-root--flush` mechanism had already been built specifically to replace this exact hack, but `OperationLogPage.css` was never updated to switch over to it when that mechanism landed — so the page was compensating a padding that could have simply been turned off via `--flush`, or (per user's choice) dropped entirely for consistency. **This is inferred from the comments in both files, not from a runtime measurement** — no live DOM inspection was done, since this was a code-cleanup finding rather than a reported visual bug; the project's `CLAUDE.md` FE UI/UX Fix Rule was still followed by asking the user for their preferred direction before writing any CSS diff, since removing a full-bleed layout is a visible UI change even without a "bug report" trigger.
- No build failures or test failures occurred during this session — `pnpm build` passed clean on every check.

## 8. Verify

```bash
cd apps/web && pnpm build
```
Expected: `tsc -b` completes with no type errors, followed by `vite build` completing with only the pre-existing (unrelated) large-chunk-size warning — no new errors or warnings introduced by any of the three changes. This was run and passed after each of the two commits in this session.

```bash
git -C apps/web/src grep -rn "FloodMapPage\|page-root--flush\|FLUSH_PAGE_PREFIXES" -- ':!../../plans' ':!../../docs'
```
Expected: no matches (confirms both the dead component and the flush mechanism are fully gone from live code).

## 9. Gotchas

- If a future page needs a genuinely full-bleed layout again, there is no `page-root--flush` shortcut anymore — it must either request shell padding be conditionally disabled again (reintroducing the route-based branching that was just removed) or handle it entirely within its own CSS without touching `AppShell`.
- `use-simulation-workbench.ts` and `workbench-validation.ts` are now the single source of truth for simulation state and validation — any new simulation feature should extend the hook rather than re-adding local state to `SimulationPage.tsx`, or the readability problem this refactor fixed will recur.
- `--pad-page` (`apps/web/src/styles.css:102`) is now the only page-spacing mechanism; a page that visually needs different spacing should override locally with normal CSS rather than reaching for margin-cancellation tricks against the shell.
