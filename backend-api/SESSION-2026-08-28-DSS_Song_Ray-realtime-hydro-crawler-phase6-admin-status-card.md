# Session 2026-08-28 — Realtime hydro crawler, Phase 6: admin crawler status card

## 1. Requirement recap

Phase 6 of the realtime-hydro-crawler plan: add a **read-only** admin-only status card showing
`crawler_run_log` health — last run per source, status, rolling 24h failure count, staleness — as
the crawler feature's *only* UI touchpoint. Explicitly out of scope: any presence on
Dashboard/Simulation/Alerts/Forecast pages, and any "run now"/write action (YAGNI, per phase scope).

## 2. How it was implemented + docs used

- **Backend read model**: `CrawlerRunLogService.latestPerSource()` uses TypeORM's
  `distinctOn(["log.source"])` + `orderBy("log.source", "ASC").addOrderBy("log.started_at", "DESC")`
  to get one row per source in a single query (Postgres `DISTINCT ON`, documented as
  Postgres-only in a code comment — the whole stack already assumes Postgres, so no portability
  cost). A second query aggregates `COUNT(*) ... GROUP BY source WHERE status='failure' AND
  started_at >= now-24h` for the rolling failure count.
- Injected the existing `CRAWLER_ADAPTERS` DI token (from phase 2) into the service so a source with
  zero runs yet still appears in the response as "not run" instead of being silently absent, and so
  an expected cron interval can be looked up per source for staleness detection.
- No `cron-parser` dependency existed in the repo and none was added. `estimateIntervalMinutes(cron)`
  only handles the two cron shapes this codebase's `crawler.config.ts` actually produces
  (`*/N * * * *` and `0 */N * * * *`), falling back to a 60-minute default otherwise — an explicit
  YAGNI tradeoff, called out in a comment rather than silently narrowing scope.
- `cronExpressionForCategory` duplicates (rather than imports) the private switch statement already
  living in `crawler.scheduler.ts` (phase 3), to avoid coupling the read-only run-log service to the
  scheduler class for one lookup. Noted in a comment as a DRY-vs-coupling tradeoff, decided in favor
  of the smaller coupling surface.
- **Backend auth**: grepped the whole `apps/api/src` for `@Controller(['"]admin` — zero matches.
  There was no existing `/admin/*` controller to copy a guard from, contrary to what the task brief
  assumed. Found the actual reusable building blocks instead: `JwtAuthGuard`
  (`apps/api/src/auth/jwt-auth.guard.ts`), `RolesGuard` + `Reflector`-based `@Roles()` metadata
  (`apps/api/src/auth/roles.guard.ts`, `roles.decorator.ts`). `crawler-admin.controller.ts` is the
  first controller in the codebase to combine `@UseGuards(JwtAuthGuard, RolesGuard)` with
  `@Roles("admin")` — every other guarded controller (e.g. `dashboard.controller.ts`) only uses bare
  `JwtAuthGuard` with no role restriction.
- **Frontend**: `apps/web/src/services/adminService.ts` (156 lines) is 100% mock data — its own
  top-of-file comment says so explicitly ("KHÔNG có backend /admin nào... toàn bộ là mock cục bộ").
  Adding a real fetch into that file would break its documented invariant, so a separate
  `crawlerAdminService.ts` was created instead, matching the real-fetch pattern already used by
  `dashboardService.ts` (fetch + `Authorization: Bearer` header + `VITE_API_BASE_URL`).
- `AdminCrawlerStatusCard.tsx` mirrors `AdminSystemStatusCard.tsx` structure exactly, reusing the
  existing `admin-card`/`admin-status-list`/`admin-badge--{ok,warn,danger}` CSS classes — no new CSS
  file needed, and the project-wide "CSS Grid, never Flexbox" rule is satisfied automatically because
  both classes were already Grid-based.
- `AdminPage.tsx` gets a second, independent `useEffect` (separate from the existing mock
  `loadAdminData()` effect) gated on `accessToken`, matching the `if (!accessToken) return` +
  `cancelled` cleanup pattern used on `DashboardPage`/`ComparisonPage`/`SimulationPage`. Fetch errors
  are swallowed to an empty array — crawler health is treated as non-critical to the rest of `/admin`.
- Status mapping: `success`→`ok`, `partial`→`warn`, `failure`→`danger`, with an `isStale` override to
  `warn`/"Trễ lịch" when `now - lastRunAt > 2x expected interval`, regardless of raw status.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/api/src/crawler/crawler-run-log.service.ts` | Write layer for `crawler_run_log` | only `record()` | adds `latestPerSource()`, `estimateIntervalMinutes()`, module-level `cronExpressionForCategory()`, `CrawlerSourceStatus` interface; constructor now also injects `CRAWLER_ADAPTERS` |
| `apps/api/src/crawler/crawler-admin.controller.ts` | Admin-only read endpoint | did not exist | `GET /admin/crawler-status`, `JwtAuthGuard`+`RolesGuard`+`@Roles("admin")`, delegates to `latestPerSource()` |
| `apps/api/src/crawler/crawler-run-log.service.spec.ts` | Unit coverage | did not exist | 2 tests: happy path (latest run + failure count mapped per adapter, unseen adapter shows null/stale), staleness threshold (2x expected interval) |
| `apps/api/src/crawler/crawler.module.ts` | Nest module wiring | no `controllers` array | adds `controllers: [CrawlerAdminController]` |
| `apps/web/src/services/crawlerAdminService.ts` | Real FE fetch for crawler status | did not exist | `fetchCrawlerStatus(accessToken)` hitting `/admin/crawler-status` |
| `apps/web/src/components/admin/AdminCrawlerStatusCard.tsx` | Admin card UI | did not exist | renders one row per source with tone/label/value mapping, reuses `AdminSystemStatusCard.css` |
| `apps/web/src/pages/AdminPage.tsx` | Admin page composition | 1 mock `useEffect`, no crawler card | 2nd `accessToken`-gated `useEffect`, `AdminCrawlerStatusCard` rendered in `admin-overview-grid` |

## 4. Code changes in detail

### 1. `latestPerSource()` read model — `apps/api/src/crawler/crawler-run-log.service.ts`

**Before:** (method did not exist; service only had `record()`)

**After:**
```ts
async latestPerSource(): Promise<CrawlerSourceStatus[]> {
  const config = crawlerConfig();

  const latestRows = await this.crawlerRunLogRepo
    .createQueryBuilder("log")
    .distinctOn(["log.source"])
    .orderBy("log.source", "ASC")
    .addOrderBy("log.started_at", "DESC")
    .getMany();

  const since = new Date(Date.now() - 24 * 60 * 60 * 1000);
  const failureCounts = await this.crawlerRunLogRepo
    .createQueryBuilder("log")
    .select("log.source", "source")
    .addSelect("COUNT(*)", "count")
    .where("log.status = :status", { status: "failure" })
    .andWhere("log.started_at >= :since", { since })
    .groupBy("log.source")
    .getRawMany<{ source: string; count: string }>();
  const failuresBySource = new Map(failureCounts.map((row) => [row.source, Number(row.count)]));

  const latestBySource = new Map(latestRows.map((row) => [row.source, row]));
  const knownSources = new Set([...this.adapters.map((a) => a.name), ...latestBySource.keys()]);

  return [...knownSources].sort().map((source) => {
    const latest = latestBySource.get(source) ?? null;
    const adapter = this.adapters.find((a) => a.name === source);
    const expectedIntervalMinutes = adapter
      ? estimateIntervalMinutes(cronExpressionForCategory(config, adapter.category))
      : FALLBACK_EXPECTED_INTERVAL_MINUTES;
    const staleAfterMs = expectedIntervalMinutes * 60 * 1000 * 2;
    const isStale = latest ? Date.now() - latest.startedAt.getTime() > staleAfterMs : true;

    return {
      source,
      lastRunAt: latest?.startedAt ?? null,
      status: latest?.status ?? null,
      recordsIngested: latest?.recordsIngested ?? null,
      failuresLast24h: failuresBySource.get(source) ?? 0,
      isStale,
    };
  });
}
```

**What changed:** new method combining two queries (one `DISTINCT ON` for latest-per-source, one
`GROUP BY` for 24h failure counts), unioned against the DI-injected adapter list so unseen sources
still get a row.

**Why:** the admin card needs "one entry per registered adapter, even if it has never run" — a plain
`SELECT ... GROUP BY source ORDER BY started_at DESC LIMIT 1` per source would require N queries or a
window function; `DISTINCT ON` gets it in one round trip, and merging with `CRAWLER_ADAPTERS` covers
the "never ran" case that a pure log-table query cannot express.

**How it behaves now:** `GET /admin/crawler-status` returns an array with exactly one object per
adapter registered in `CRAWLER_ADAPTERS`, each carrying its latest run status, 24h failure count, and
a computed `isStale` flag.

### 2. Cron-interval estimate without a cron parser — `apps/api/src/crawler/crawler-run-log.service.ts`

**Before:** (did not exist)

**After:**
```ts
/** Minimal interval estimate for the two cron shapes actually used by `crawler.config.ts`
 * (`*/N * * * *` and `0 */N * * * *`) — good enough for "is this run stale" checks, not a
 * general cron parser (YAGNI, no new dependency for this). */
function estimateIntervalMinutes(cron: string): number {
  const fields = cron.trim().split(/\s+/);
  const [minute, hour] = fields;
  const minuteStep = minute.match(/^\*\/(\d+)$/);
  if (minuteStep) {
    return Number(minuteStep[1]);
  }
  const hourStep = hour.match(/^\*\/(\d+)$/);
  if (hourStep && minute === "0") {
    return Number(hourStep[1]) * 60;
  }
  return FALLBACK_EXPECTED_INTERVAL_MINUTES;
}
```

**What changed:** a hand-written, two-pattern cron interpreter instead of a general cron library.
**Why:** the repo's `crawler.config.ts` only ever produces `*/N * * * *` or `0 */N * * * *` strings
(checked, not assumed — these are the only two shapes referenced by `crawlerConfig()`); pulling in a
full cron-parser dependency for two regexes would violate YAGNI.
**How it behaves now:** any cron string outside those two shapes silently falls back to a
conservative 60-minute expected interval rather than throwing, so staleness detection degrades
gracefully instead of crashing the admin endpoint.

### 3. Admin-only guarded controller — `apps/api/src/crawler/crawler-admin.controller.ts` (new)

**Before:** no `/admin/*` controller existed anywhere in `apps/api/src`.

**After:**
```ts
@ApiTags("admin")
@ApiBearerAuth()
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles("admin")
@Controller("admin/crawler-status")
export class CrawlerAdminController {
  constructor(private readonly crawlerRunLogService: CrawlerRunLogService) {}

  @Get()
  @ApiOperation({ summary: "Latest crawler_run_log row per source, plus 24h failure count (admin-only)" })
  @ApiResponse({ status: 200, description: "One entry per registered adapter source" })
  getCrawlerStatus() {
    return this.crawlerRunLogService.latestPerSource();
  }
}
```

**What changed:** new controller composing `JwtAuthGuard` (populates `req.user`) with `RolesGuard` +
`@Roles("admin")` (existing but previously unused-together auth primitives).
**Why:** the task needed an admin-only route, and the codebase already had the guard/decorator
building blocks — this reuses them instead of hand-rolling a new auth check.
**How it behaves now:** unauthenticated requests get a 401 from `JwtAuthGuard`; authenticated
non-admin users get rejected by `RolesGuard`; only admins reach `latestPerSource()`.

### 4. Separate real-fetch service instead of extending the mock one — `apps/web/src/services/crawlerAdminService.ts` (new)

**Before:** all `/admin` page data came from `apps/web/src/services/adminService.ts`, whose own
header comment states everything there is local mock data.

**After:**
```ts
export async function fetchCrawlerStatus(accessToken: string): Promise<CrawlerSourceStatus[]> {
  const response = await fetch(`${API_BASE_URL}/admin/crawler-status`, {
    headers: { Authorization: `Bearer ${accessToken}` },
  });

  if (!response.ok) {
    throw new Error(`Crawler status request failed: ${response.status}`);
  }
  return response.json() as Promise<CrawlerSourceStatus[]>;
}
```

**What changed:** a brand-new file rather than a function added to `adminService.ts`.
**Why:** mixing one real endpoint into a file whose documented contract is "100% mock" would break
that contract for every other consumer of `adminService.ts` and mislead future readers of that
comment.
**How it behaves now:** the crawler card fetches real backend data while the rest of `/admin`
continues to render mock data unchanged, with a visible seam (a separate file) marking which is which.

### 5. Admin page wiring — `apps/web/src/pages/AdminPage.tsx`

**Before:**
```ts
useEffect(() => {
  let cancelled = false;
  loadAdminData().then((result) => { ... });
  return () => { cancelled = true; };
}, []);
```
(only the mock-data effect existed; no crawler card rendered)

**After:**
```ts
useEffect(() => {
  if (!accessToken) return;
  let cancelled = false;
  fetchCrawlerStatus(accessToken)
    .then((rows) => {
      if (!cancelled) setCrawlerSources(rows);
    })
    .catch(() => {
      // Card falls back to its own empty state; crawler health is non-critical to /admin.
    });
  return () => {
    cancelled = true;
  };
}, [accessToken]);
```
and in the render tree:
```ts
<AdminSystemStatusCard metrics={data.systemMetrics} />
<AdminCrawlerStatusCard sources={crawlerSources} />
```

**What changed:** a second, independent `useEffect` gated on `accessToken`, and one new card in
`admin-overview-grid`.
**Why:** matches the existing `accessToken`-gated fetch pattern used on Dashboard/Comparison/
Simulation pages rather than inventing a new one; keeping it separate from the mock-data effect means
a crawler-endpoint failure can't block the rest of the (mock) admin page from loading.
**How it behaves now:** the card renders with real data once `accessToken` is available and the
fetch succeeds; on failure it silently stays at its empty-state (`sources.length === 0` → "Chưa có
dữ liệu"), never breaking page load.

## 5. How to find this again

- Route: `GET /admin/crawler-status`
- Backend: `CrawlerRunLogService.latestPerSource`, `CrawlerAdminController`,
  `estimateIntervalMinutes`, `cronExpressionForCategory`
- Frontend: `fetchCrawlerStatus`, `AdminCrawlerStatusCard`, `crawlerSources` state in `AdminPage.tsx`
- Auth pattern to copy for future admin-only routes: `JwtAuthGuard` + `RolesGuard` + `@Roles("admin")`
  (`apps/api/src/auth/jwt-auth.guard.ts`, `roles.guard.ts`, `roles.decorator.ts`)

## 6. Concepts introduced

- **`DISTINCT ON` (Postgres)**: returns the first row per group according to `ORDER BY`, in one
  query — used here instead of a window function or N+1 queries to get "latest run per source."
  Postgres-specific; needed because the alternative (querying per-adapter in a loop) doesn't scale
  and TypeORM has no cross-dialect equivalent.
- **NestJS `RolesGuard` + `Reflector`**: a guard that reads metadata attached by a custom
  `@Roles(...)` decorator (via `Reflector.getAllAndOverride`) and compares it against
  `req.user.roles`. Needed because this was the first route in the codebase actually requiring
  role-based (not just authenticated) access.
- **Postgres session-level context for staleness**: not new to this session (built in phase 3), but
  reused here — the "2x expected interval" heuristic depends on the cron config already validated at
  boot in phase 2/3.

## 7. Where it got stuck

- **Symptom:** task brief assumed an existing `/admin/*` controller to copy a guard pattern from.
  **Cause (directly observed):** `grep -r '@Controller(['"'"'"]admin' apps/api/src` returned zero
  matches — no such controller exists anywhere in the API.
  **Fix:** searched for the underlying auth primitives instead of a controller example, found
  `JwtAuthGuard`/`RolesGuard`/`@Roles()` already implemented and unused together, and composed them
  directly on the new controller. This satisfies the task's real intent ("reuse existing admin auth,
  don't hand-roll") even though the literal artifact (an existing admin controller) didn't exist.

- **Symptom:** risk of silently corrupting the `/admin` page's documented "100% mock" contract.
  **Cause (directly observed):** read `apps/web/src/services/adminService.ts` in full (156 lines);
  its header comment explicitly states there is no real `/admin` backend and everything is local
  mock data.
  **Fix:** created a new, separate `crawlerAdminService.ts` file instead of adding a function to
  `adminService.ts`, keeping the mock-vs-real boundary explicit and matching the existing pattern in
  `dashboardService.ts` for real fetches.

- **Symptom:** `pnpm --filter @dss/web build` reported 3 TypeScript errors.
  **Cause (directly observed via `git status --short` on the exact reported file paths):**
  `forecast-summary-card.tsx` (unused variable) and `AppMapCanvas.tsx` (two tuple-type mismatches)
  showed no diff for this session — confirmed pre-existing, unrelated to this change.
  **Fix:** did not touch those files; instead verified the actually-changed files with a scoped
  `npx tsc --noEmit -p tsconfig.json` filtered to the new/modified paths, which came back clean.

- **Symptom:** `pnpm --filter @dss/api test -- crawler` showed one failing suite,
  `app.controller.spec.ts`, `SyntaxError: Unexpected token 'export'` from `packages/shared/src/index.ts`.
  **Cause (inferred, evidenced by the error itself):** Jest in this monorepo isn't configured to
  transform the `@dss/shared` workspace package (it ships ESM `export` syntax Jest's default CJS
  transform can't parse) — a pre-existing Jest/workspace-package config gap, not something this
  session's crawler code touches.
  **Fix:** none applied (out of scope); confirmed all 20 other suites (90 tests, including the 2 new
  crawler-run-log tests) passed, isolating the failure to that one unrelated suite.

- **Symptom:** could not exercise the endpoint or card against a real database or dev server.
  **Cause (directly observed):** `psql` not on PATH in this sandbox, no API/web dev servers running.
  **Fix:** compensated with `nest build` (clean), a new unit spec covering the happy path and
  staleness threshold (both passing), and a scoped FE typecheck — flagged explicitly here as an
  unverified gap rather than glossed over, consistent with prior phases in this plan.

## 8. Verify

- `pnpm --filter @dss/api build` → clean, no errors.
- `pnpm --filter @dss/api test -- crawler-run-log` → 2/2 tests passed
  (`CrawlerRunLogService.latestPerSource`: happy-path mapping, staleness threshold).
- `pnpm --filter @dss/api test -- crawler` → 90/90 crawler-related tests passed across the suite;
  the one failing suite (`app.controller.spec.ts`) is the pre-existing, unrelated `@dss/shared`
  Jest-transform gap described above.
- `npx tsc --noEmit -p tsconfig.json` scoped to the changed FE files (`crawlerAdminService.ts`,
  `AdminCrawlerStatusCard.tsx`, `AdminPage.tsx`) → zero errors.

## 9. Gotchas

- `latestPerSource()` uses `DISTINCT ON`, a Postgres-only feature — porting this stack to another
  RDBMS would require rewriting this query (e.g. a window function + `ROW_NUMBER() = 1` filter).
- `estimateIntervalMinutes` only understands two cron shapes; if `crawler.config.ts` ever gains a
  cron pattern outside `*/N * * * *` / `0 */N * * * *` (e.g. day-of-week or specific-hour patterns),
  staleness detection will silently fall back to a 60-minute default instead of erroring — worth
  revisiting if more adapters with unusual schedules are added.
- `cronExpressionForCategory` is a second, independent copy of the switch statement in
  `crawler.scheduler.ts`. If a new adapter `category` is ever added, both switch statements need to
  be updated — there is no compiler-enforced link between them (a missing `case` here returns
  `undefined`, silently defaulting the estimate). A future refactor could extract a shared helper if
  a third or fourth consumer of this mapping appears (currently only 2 — not yet worth the coupling
  per DRY-vs-coupling call made this session).
- The FE fetch effect swallows all errors silently; if `/admin/crawler-status` starts returning
  meaningful error bodies (e.g. distinguishing "no admin role" from "server error"), this will need
  surfacing to the user instead of a blanket empty state.
