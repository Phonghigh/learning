# Session 2026-09-02 — Crawler HTTP audit log (retention + unhealthy-source detection)

## 1. Requirement recap

Implement a 6-phase plan (`plans/260902-1042-crawler-http-audit-log/`) that adds full HTTP
request/response auditing for every crawler run: a new `crawler_http_log` table, an
`AsyncLocalStorage`-based `loggedFetch()` wrapper that captures each call with header/body
redaction and truncation, wiring into `crawler.scheduler.ts`'s `runAdapter`, a
`consecutiveFailures`/`isUnhealthy` field surfaced on the admin crawler-status dashboard (backend +
React FE), a daily retention cron (30 days, cascade delete), and full Jest coverage. Origin: a real
incident where 435 crawler runs failed silently because `crawler_run_log` only stored a status
string, not what actually happened on the wire.

## 2. How it was implemented + docs used

The plan/phase files (already researched and approved before this session) fixed the design, so
this session was pure implementation, not re-design. Key decisions carried from the plan:

- **AsyncLocalStorage (approach A)** over changing `ExternalDataSourceAdapter.fetch()`'s
  signature — every adapter keeps returning `Promise<RawReading[]>`; the ALS store threads the
  log-entry array through the retry/lock chain implicitly.
- **Raw SQL window function** for `consecutiveFailuresBySource()` instead of TypeORM
  `QueryBuilder` — the existing `crawler-run-log.service.spec.ts` fakes `createQueryBuilder` by
  call-count order (`callCount === 1 ? distinctBuilder : groupBuilder`); adding a third
  `createQueryBuilder()` call would have broken that fragile mock. Raw `repo.query()` sidesteps it
  entirely and was accepted since the service already leans on Postgres-only features
  (`DISTINCT ON`).
- **Reverse import crawler → forecast**: `open-meteo-ensemble.client.ts` (not the
  `open-meteo-rainfall` adapter itself) makes the actual HTTP call, so logging it required
  `apps/api/src/forecast/open-meteo-ensemble.client.ts` to import `loggedFetch` from
  `crawler/logging/`. Accepted as a reasonable coupling trade-off rather than leaving that source
  unlogged.
- **Migration style**: raw-SQL `QueryRunner.query()` calls, matching this codebase's existing
  migration convention (not the `Table`/`QueryRunner` builder API).
- Existing code reused: `stripQueryStrings` (previously private to
  `crawler-run-log.service.ts`) was extracted to a new shared
  `crawler/logging/http-log-redaction.ts` so `loggedFetch` could reuse it (DRY) instead of
  duplicating query-string stripping logic.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/api/src/crawler/crawler.scheduler.ts` | Runs one adapter tick | lock+retry+ingest directly in `try` | wrapped in `httpLogContext.run(entries, async () => {...})`; persists `httpLogEntries` via `recordHttpLogs` in `finally` |
| `apps/api/src/crawler/crawler-run-log.service.ts` | Run-log persistence + dashboard query | `record()` returned `void`; 2 parallel `createQueryBuilder` queries; local `stripQueryStrings` | `record()` returns the saved `CrawlerRunLog`; added `recordHttpLogs()`, `consecutiveFailuresBySource()` (raw SQL window fn), `RETENTION_DAYS`/`CONSECUTIVE_FAILURE_THRESHOLD` constants; imports `stripQueryStrings` from shared module |
| `apps/api/src/crawler/logging/http-log-context.ts` | ALS store | (new) | `httpLogContext: AsyncLocalStorage<HttpLogEntry[]>` + `HttpLogEntry` interface |
| `apps/api/src/crawler/logging/http-log-redaction.ts` | Redaction/truncation helpers | (new, `stripQueryStrings` lived in `crawler-run-log.service.ts`) | `stripQueryStrings`, `redactHeaders` (denylist: authorization/cookie/set-cookie/x-api-key), `truncateBody` (256KB cap) |
| `apps/api/src/crawler/logging/logged-fetch.ts` | Fetch wrapper | (new) | `loggedFetch()` — records into active ALS store or no-ops to plain `fetch` |
| `apps/api/src/crawler/entities/crawler-http-log.entity.ts` | TypeORM entity | (new) | maps to `crawler_http_log` |
| `apps/api/src/database/migrations/20260902110000-AddCrawlerHttpLog.ts` | DB migration | (new) | creates `crawler_http_log` with FK `run_log_id -> crawler_run_log(id) ON DELETE CASCADE` + 2 indexes |
| `apps/api/src/crawler/crawler-retention.service.ts` | Retention cron | (new) | daily `@Cron("0 3 * * *")` deletes `crawler_run_log` rows older than `RETENTION_DAYS` (children cascade) |
| `apps/api/src/crawler/crawler.module.ts` | Module wiring | — | registers `CrawlerHttpLog` entity + `CrawlerRetentionService` |
| `apps/api/src/forecast/open-meteo-ensemble.client.ts` | Open-Meteo ensemble HTTP call | `fetch(url.toString())` | `loggedFetch(url.toString())` |
| `apps/web/src/components/admin/AdminCrawlerStatusCard.tsx` + `crawlerAdminService.ts` | Admin dashboard | no unhealthy indicator | red `danger` badge + "Lỗi liên tiếp (N)" label when `isUnhealthy` |

## 4. Code changes in detail

### 1. `runAdapter`'s `try` body wrapped in an ALS context — `apps/api/src/crawler/crawler.scheduler.ts`

**Before:**
```ts
try {
  const readings = await this.advisoryLockService.withLock(adapter.name, () =>
    pRetry(() => adapter.fetch(), {
      ...RETRY_OPTIONS,
      onFailedAttempt: (error) => {
        this.logger.warn(
          `[${adapter.name}] attempt ${error.attemptNumber} failed, ${error.retriesLeft} retries left: ${error.message}`,
        );
      },
    }),
  );

  if (readings === null) {
    // Another instance already holds the lock for this adapter; this
    // instance did nothing, so it writes no run-log row.
    lockSkipped = true;
    return;
  }

  const { inserted, skipped } = await this.ingestService.persist(readings);
  recordsIngested = inserted;
  if (skipped > 0) {
    status = "partial";
  }
} catch (error) {
  ...
} finally {
  if (!lockSkipped) {
    await this.crawlerRunLogService.record({ ... });
  }
}
```

**After:**
```ts
const httpLogEntries: HttpLogEntry[] = [];

try {
  await httpLogContext.run(httpLogEntries, async () => {
    const readings = await this.advisoryLockService.withLock(adapter.name, () =>
      pRetry(() => adapter.fetch(), { ...RETRY_OPTIONS, onFailedAttempt: (error) => { ... } }),
    );

    if (readings === null) {
      // `return` here only exits this ALS callback (not `runAdapter`) — the
      // `finally` block below still runs and is guarded by `lockSkipped`.
      lockSkipped = true;
      return;
    }

    const { inserted, skipped } = await this.ingestService.persist(readings);
    recordsIngested = inserted;
    if (skipped > 0) {
      status = "partial";
    }
  });
} catch (error) {
  ...
} finally {
  if (!lockSkipped) {
    const runLog = await this.crawlerRunLogService.record({ ... });
    try {
      await this.crawlerRunLogService.recordHttpLogs(runLog.id, httpLogEntries);
    } catch (e) {
      this.logger.warn(`[${adapter.name}] failed to persist crawler_http_log rows: ${e instanceof Error ? e.message : String(e)}`);
    }
  }
}
```

**What changed:** the entire lock+retry+ingest body moved from the outer `try` into an
`async () => {}` callback passed to `httpLogContext.run(httpLogEntries, ...)`; `httpLogEntries` is
pre-allocated as `const [] : HttpLogEntry[]` before the `try` (not read back from `getStore()`
later) so the reference survives after the callback returns; `record()`'s result is now captured
(`const runLog = await ...`) and immediately used to call the new `recordHttpLogs(runLog.id,
httpLogEntries)`, itself wrapped in its own try/catch so a logging failure never turns a
successful crawl into a crash.

**Why:** without an ALS context open around `adapter.fetch()`, `loggedFetch()` (called deep inside
adapter code) has no store to push entries into and silently no-ops — the audit table would stay
empty. Opening the context around the *lock+retry* path (not just `pRetry`) ensures entries from
every retry attempt, and from any nested `Promise.all` fetches inside an adapter, land in the same
array.

**How it behaves now:** every HTTP attempt made by an adapter during a tick — success, retry, or
final failure — is captured, and after the run-log row is written, its `id` is used to bulk-insert
the matching `crawler_http_log` rows via a single `insert()` call.

### 2. `record()` return type + new query methods — `apps/api/src/crawler/crawler-run-log.service.ts`

**Before:**
```ts
async record(input: RecordRunInput): Promise<void> {
  const sanitizedMessage = input.errorMessage
    ? stripQueryStrings(input.errorMessage).slice(0, MAX_ERROR_MESSAGE_LENGTH)
    : null;

  await this.crawlerRunLogRepo.save(
    this.crawlerRunLogRepo.create({ ... }),
  );
}
```

**After:**
```ts
async record(input: RecordRunInput): Promise<CrawlerRunLog> {
  const sanitizedMessage = input.errorMessage
    ? stripQueryStrings(input.errorMessage).slice(0, MAX_ERROR_MESSAGE_LENGTH)
    : null;

  return await this.crawlerRunLogRepo.save(
    this.crawlerRunLogRepo.create({ ... }),
  );
}

async recordHttpLogs(runLogId: string, entries: HttpLogEntry[]): Promise<void> {
  if (entries.length === 0) return;
  const rows = entries.map((entry, index) => ({ runLogId, sequence: entry.sequence ?? index, ... }));
  await this.crawlerHttpLogRepo.insert(rows);
}
```

**What changed:** `record()`'s return type changed from `Promise<void>` to
`Promise<CrawlerRunLog>` (returns the saved entity so the caller has the new row's `id`); added
`recordHttpLogs()` using `insert()` (bulk, no per-row round trip) rather than `save()`.

**Why:** `crawler.scheduler.ts` needs `runLog.id` as the foreign key for the HTTP log rows it is
about to insert — the previous `void` return made that impossible without a second query.

**How it behaves now:** one `record()` call now gives the caller everything needed to also write
the child audit rows in the same tick, with a single bulk `INSERT`.

### 3. `latestPerSource()` gains a third parallel query — `apps/api/src/crawler/crawler-run-log.service.ts`

**Before:**
```ts
const latestRows = await this.crawlerRunLogRepo.createQueryBuilder("log").distinctOn(...).getMany();
const since = new Date(Date.now() - 24 * 60 * 60 * 1000);
const failureCounts = await this.crawlerRunLogRepo.createQueryBuilder("log").select(...).getRawMany();
```

**After:**
```ts
const since = new Date(Date.now() - 24 * 60 * 60 * 1000);
const [latestRows, failureCounts, consecutiveFailuresBySource] = await Promise.all([
  this.crawlerRunLogRepo.createQueryBuilder("log").distinctOn(["log.source"]).orderBy(...).getMany(),
  this.crawlerRunLogRepo.createQueryBuilder("log").select("log.source", "source").addSelect("COUNT(*)", "count").where(...).groupBy(...).getRawMany(),
  this.consecutiveFailuresBySource(),
]);
```
and per-source mapping gains:
```ts
const consecutiveFailures = consecutiveFailuresBySource.get(source) ?? 0;
return {
  ...,
  consecutiveFailures,
  isUnhealthy: consecutiveFailures >= CONSECUTIVE_FAILURE_THRESHOLD || isStale,
};
```

**What changed:** the two sequential `await`s became a `Promise.all` of three queries (added
`consecutiveFailuresBySource()`); `CrawlerSourceStatus` gained `consecutiveFailures: number` and
`isUnhealthy: boolean`.

**Why:** the dashboard needs to flag sources that are consistently failing, not just currently
stale, per the plan's phase-04 requirement.

**How it behaves now:** a source with 3+ unbroken failures at the head of its history (bounded to
`RETENTION_DAYS`) is flagged `isUnhealthy`, surfaced on the FE as a red badge.

### 4. New window-function query — `consecutiveFailuresBySource()` in `crawler-run-log.service.ts`

```sql
SELECT source, COUNT(*) AS count
FROM (
  SELECT source, status,
         ROW_NUMBER() OVER (PARTITION BY source ORDER BY started_at DESC) AS rn,
         SUM(CASE WHEN status <> 'failure' THEN 1 ELSE 0 END)
           OVER (PARTITION BY source ORDER BY started_at DESC
                 ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS non_failures_before
  FROM crawler_run_log
  WHERE started_at >= now() - interval '30 days'
) t
WHERE status = 'failure' AND non_failures_before = 0
GROUP BY source
```

**What changed:** new private method, raw `repo.query()` instead of `QueryBuilder`.

**Why:** computing "unbroken failure streak counted backwards from the most recent run" per source
needs a running aggregate (`SUM(...) OVER (... ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`),
which is awkward via TypeORM's `QueryBuilder`. Using raw SQL also avoided a third
`createQueryBuilder()` call that would have broken `crawler-run-log.service.spec.ts`'s
call-count-based mock (see section 7).

**How it behaves now:** one round trip returns a `source -> streak length` map used directly by
`latestPerSource()`, instead of N+1 per-source queries.

### 5. `loggedFetch()` — new file `apps/api/src/crawler/logging/logged-fetch.ts`

```ts
export async function loggedFetch(input: RequestInfo | URL, init?: RequestInit): Promise<Response> {
  const store = httpLogContext.getStore();
  if (!store) {
    return fetch(input, init);
  }
  ...
  const response = await fetch(input, init);
  const text = await response.clone().text();   // clone so caller's own .json()/.text() still works
  ...
  store.push(entry);
  return response;
}
```

**What changed:** new drop-in `fetch` replacement.

**Why:** needed a single call site adapters and the forecast client could swap `fetch()` for
without changing anything else about how they consume the response.

**How it behaves now:** with an active ALS store it records method/URL/redacted headers/body,
status, redacted response headers, truncated response body, duration, and (on throw) an error
message, then pushes the entry and re-throws; with no active store it degrades to plain `fetch`
transparently — this is what lets `song-ray-reservoir-info-sync.service.ts` and a backfill script
keep calling the same underlying fetch path outside a crawler tick without crashing.

### 6. `crawler_http_log` migration — `apps/api/src/database/migrations/20260902110000-AddCrawlerHttpLog.ts`

New raw-SQL migration creating the table with `response_status`/`response_headers`/`response_body`
all nullable (a request can fail before any response arrives), plus
`CONSTRAINT "FK_crawler_http_log_run_log" FOREIGN KEY ("run_log_id") REFERENCES
"crawler_run_log"("id") ON DELETE CASCADE` and two indexes (`run_log_id`, `created_at`).

**Why:** the FK cascade is what lets `CrawlerRetentionService.purgeOldRuns()` delete only the
parent row and rely on Postgres to clean up children — no separate delete needed, avoiding
orphan-vs-ordering bugs.

### 7. Retention cron — new `apps/api/src/crawler/crawler-retention.service.ts`

```ts
@Cron("0 3 * * *", { name: "crawler-http-log-retention" })
async handleRetention(): Promise<void> {
  if (!crawlerConfig().enabled) return;
  try {
    await this.purgeOldRuns();
  } catch (error) {
    this.logger.error(`Crawler log retention failed: ...`);
  }
}

async purgeOldRuns(): Promise<number> {
  const cutoff = new Date(Date.now() - RETENTION_DAYS * 24 * 60 * 60 * 1000);
  const result = await this.crawlerRunLogRepo.delete({ startedAt: LessThan(cutoff) });
  return result.affected ?? 0;
}
```

**Why:** `RETENTION_DAYS` is imported from `crawler-run-log.service.ts` (not redefined) so the
consecutive-failure scan window and the actual retention horizon can never drift apart. No
advisory lock: the delete is idempotent, so two instances racing on it is harmless.

**How it behaves now:** every day at 03:00, run-log rows (and their cascaded HTTP logs) older than
30 days are purged; a failure is logged, not thrown, so a bad cron tick can't crash the process.

## 5. How to find this again

- `httpLogContext` / `HttpLogEntry` — `apps/api/src/crawler/logging/http-log-context.ts`
- `loggedFetch` — `apps/api/src/crawler/logging/logged-fetch.ts`, and its 2 call sites:
  `apps/api/src/forecast/open-meteo-ensemble.client.ts` and any `songRayFetch`-based adapter
- `redactHeaders` / `truncateBody` / `stripQueryStrings` — `apps/api/src/crawler/logging/http-log-redaction.ts`
- `consecutiveFailuresBySource`, `RETENTION_DAYS`, `CONSECUTIVE_FAILURE_THRESHOLD` — `apps/api/src/crawler/crawler-run-log.service.ts`
- `CrawlerRetentionService.purgeOldRuns` — `apps/api/src/crawler/crawler-retention.service.ts`
- `crawler_http_log` table — `apps/api/src/database/migrations/20260902110000-AddCrawlerHttpLog.ts`, `apps/api/src/crawler/entities/crawler-http-log.entity.ts`
- regression test for the closure-return risk: grep `"regression: closure return"` in `apps/api/src/crawler/crawler.scheduler.spec.ts`
- FE unhealthy badge: grep `"Lỗi liên tiếp"` in `apps/web/src/components/admin/AdminCrawlerStatusCard.tsx`

## 6. Concepts introduced

- **`AsyncLocalStorage`** (`node:async_hooks`): a Node API that lets a value be implicitly carried
  across an `async`/`await` call chain — including through retries and nested promises — without
  passing it as a function argument at every level. Used here so `loggedFetch()`, called deep
  inside adapter code, can find the current run's log-entry array without
  `ExternalDataSourceAdapter.fetch()`'s signature having to change to accept one.
- **Postgres window functions** (`ROW_NUMBER() OVER (...)`, `SUM(...) OVER (... ROWS BETWEEN
  UNBOUNDED PRECEDING AND CURRENT ROW)`): compute a per-row running value across a partition
  (here, per `source`, ordered by recency) in a single query pass. Used to count "unbroken failure
  streak from the most recent row backwards" without an N+1 query per source.

## 7. Where it got stuck

- **Symptom (risk flagged before it became a bug):** wrapping `runAdapter`'s `try` body in
  `httpLogContext.run(entries, async () => { ... if (readings === null) { lockSkipped = true;
  return; } ... })` turns the original `return` (which used to exit the whole `try` block of
  `runAdapter`) into a `return` that only exits the ALS callback. **Cause:** this is a classic
  closure-scoping footgun — if any code had existed *after* that block in the original `try`, the
  new version would silently start executing it instead of skipping to `finally`, changing
  behavior with no compile error. In this specific case it's harmless because nothing followed
  that block in the original code, and the outer `finally` is separately guarded by the
  `lockSkipped` flag. **Fix:** rather than trust that reasoning alone, a dedicated regression test
  was added (`does not run ingest.persist when the lock returns null (regression: closure
  return)` in `crawler.scheduler.spec.ts`) that asserts `ingestService.persist` and
  `recordHttpLogs` are never called when the lock returns `null`, so any future refactor that
  reintroduces trailing code after that block would be caught by a failing test instead of a
  silent behavior change.

- **Symptom:** adding a third failure-count query to `latestPerSource()` risked breaking
  `crawler-run-log.service.spec.ts`. **Cause:** the existing spec's fake mocks
  `createQueryBuilder` by call-count order (`callCount === 1 ? distinctBuilder : groupBuilder`) —
  a hardcoded sequence, not a per-query-shape match. A third `createQueryBuilder()` call for
  consecutive failures would have received whichever fake builder the counter landed on, likely
  returning wrong data or throwing. **Fix:** implemented `consecutiveFailuresBySource()` with raw
  `repo.query()` instead of `createQueryBuilder()`, sidestepping the fragile mock entirely rather
  than trying to extend it to handle a third call shape.

- **Symptom:** two verification steps from the plan's phase files (a manual `docker exec ... psql`
  inspection of `\d crawler_http_log` and the cascade-delete boundary, plus a
  `migration:revert` round-trip) could not be completed. **Cause:** the Bash tool's sandbox/auto
  mode classifier blocked the `docker exec` + `psql` commands outright ("Blocked by classifier").
  `migration:revert` was separately skipped by choice, not blocked, to avoid a destructive DB
  operation mid-session without an easy way to re-verify state afterward via psql. **Fix:** relied
  instead on the migration's own successful `migration:run` output (table + FK + 2 indexes logged)
  and the automated Jest specs — in particular `crawler-retention.service.spec.ts`'s boundary test
  using fake timers to simulate the 30-day cutoff — as verification evidence, and explicitly noted
  in the phase files that manual psql/revert checks were skipped due to sandbox restrictions, not
  forgotten. Note: a code comment in `crawler-run-log.service.ts` above
  `consecutiveFailuresBySource()` says the SQL was "verified against real crawler_run_log data in
  psql" — that claim could not be independently re-confirmed in this session given the same
  sandbox restriction, so treat it as inherited context from the plan rather than evidence
  gathered this session.

- **Pre-existing, unrelated failures (ruled out as this session's responsibility):**
  `apps/api/src/app.controller.spec.ts` fails to parse under Jest because
  `packages/shared/src/index.ts` uses `export` syntax Jest's default transform doesn't handle for
  that workspace package (monorepo `transformIgnorePatterns` misconfiguration), and
  `operation-log.service.spec.ts` has a pre-existing TS2741 error (missing `externalId` field).
  Confirmed via `git status`/`git log` on those specific paths showing no changes from this
  session — both predate it.

## 8. Verify

- `pnpm --filter api migration:run` — succeeded; logged the full `CREATE TABLE crawler_http_log`
  + FK + 2 `CREATE INDEX` statements.
- `pnpm --filter api exec tsc --noEmit` — clean except the 2 pre-existing unrelated errors noted
  above.
- `pnpm --filter web exec tsc --noEmit` — clean.
- `cd apps/api && npx jest src/crawler` — 76/76 passed (crawler-only suite run standalone first to
  isolate).
- `npx jest src/forecast` — 21/21 passed.
- `npx jest` (full suite) — 153/153 actual tests passing across 30 passing suites; 1 suite
  (`app.controller.spec.ts`) fails to transform for the pre-existing, unrelated reason above.
- `pnpm --filter api build` — clean Nest build.
- `pnpm --filter api lint` — no lint script exists for `apps/api`
  (`ERR_PNPM_RECURSIVE_RUN_NO_SCRIPT`); step skipped as not applicable.

## 9. Gotchas

- **Closure-return footgun**: any future change that adds code *after* the
  `httpLogContext.run(entries, async () => { ... })` callback body in `runAdapter`'s `try` block
  must double-check whether an early `return` inside that callback was meant to skip that new
  code too — it won't, it only exits the callback. The regression test only covers the current
  shape; a differently-placed `return` needs its own test.
- **`RETENTION_DAYS` is a single source of truth** shared between `CrawlerRetentionService` and
  the consecutive-failure scan window in `crawler-run-log.service.ts`. Changing the retention
  window in one place without checking the other would silently desync "how long logs are kept"
  from "how far back streaks are counted."
- **`recordHttpLogs` failures are swallowed by design** (logged, not thrown) so a broken audit
  insert never fails a successful crawl — but this means a persistent DB issue with
  `crawler_http_log` (e.g. a broken migration) would show up only in logs, never in monitoring
  that watches `crawler_run_log` status. Worth pairing with a log-based alert if this table starts
  silently failing to receive rows.
- **`REDACTED_HEADERS` is a denylist**, not an allowlist — a new adapter that introduces a novel
  secret header name (not `authorization`/`cookie`/`set-cookie`/`x-api-key`) will leak its value
  into `crawler_http_log` until someone adds it to the set. Any new adapter with a custom auth
  header must add it here.
- **Manual psql verification of the cascade delete and the 29/31-day retention boundary was never
  actually run against a live DB in this session** — it was blocked by sandbox restrictions and
  substituted with automated tests using fake timers. If retention behavior is ever suspect in
  production, that manual check is still outstanding.
- **`loggedFetch`'s no-op fallback is silent** — if `httpLogContext.run(...)` is ever accidentally
  removed or misplaced around a future call site, the audit table simply stops receiving rows for
  that path with no error anywhere; there's no assertion that a context was expected.
