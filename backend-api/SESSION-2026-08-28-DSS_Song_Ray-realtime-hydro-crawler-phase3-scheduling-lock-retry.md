# Session 2026-08-28 — Realtime hydro crawler, Phase 3: scheduling, advisory lock, retry

## 1. Requirement recap

Phase 3 of `plans/260828-0909-realtime-hydro-crawler/phase-03-scheduling-lock-and-retry.md`: turn
phase 2's inert `CRAWLER_ADAPTERS` skeleton into a live scheduler — one cron job per registered
adapter, guarded so two instances of the app never run the same adapter concurrently (multi-instance
safety), with retry/backoff around the network fetch so a single flaky HTTP call doesn't kill the
whole tick. Persistence into `crawler_run_log` is explicitly out of scope (phase 4).

## 2. How it was implemented + docs used

Two new services, wired into the existing `CrawlerModule`:

- **`AdvisoryLockService.withLock<key, fn>`** — wraps Postgres session-level advisory locks
  (`pg_try_advisory_lock` / `pg_advisory_unlock`) around `hashtext(key)`, using one dedicated
  `QueryRunner` for the entire lock lifetime. Chosen over a DB-row-based lock table because it needs
  no schema, no cleanup of stale rows, and Postgres releases the lock automatically if the holding
  connection dies (crash-safe by construction) — a plain "SELECT ... FOR UPDATE" row lock or a
  hand-rolled `locks` table would need manual expiry/cleanup logic instead.
- **`CrawlerScheduler`** (`OnModuleInit` / `OnModuleDestroy`) — reads the `CRAWLER_ADAPTERS` DI array
  from phase 2, and for each adapter registers one dynamic `CronJob` (from the `cron` package) via
  `SchedulerRegistry.addCronJob`. Dynamic registration was required, not `@nestjs/schedule`'s
  `@Cron()` decorator, because the number of adapters and their cron expressions are only known at
  runtime (from `CRAWLER_ADAPTERS` + `crawlerConfig()`), not at class-definition time. Each tick runs
  `adapter.fetch()` through `p-retry` (4 retries, factor 2, 1s base) inside `advisoryLockService
  .withLock()`, and the whole thing is wrapped in try/catch so a cron tick can never produce an
  unhandled rejection that could crash the process.
- **`ScheduleModule.forRoot()`** registered locally inside `CrawlerModule` (not `AppModule`) to keep
  cron machinery scoped to the one feature that needs it.

No existing code needed reshaping; `crawlerConfig()` (phase 2) is reused as-is via direct function
call rather than promoted into NestJS's `ConfigService` DI system, matching how it was already
written.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/api/src/crawler/advisory-lock.service.ts` | Multi-instance mutex per adapter key | did not exist | `AdvisoryLockService.withLock<T>(key, fn)` using a dedicated `QueryRunner`, releases lock+connection in `finally` |
| `apps/api/src/crawler/advisory-lock.service.spec.ts` | Unit coverage for lock lifecycle | did not exist | 4 tests: success path, single-connection-for-both-calls, lock-held-elsewhere skip, release-on-throw |
| `apps/api/src/crawler/crawler.scheduler.ts` | Registers + runs one cron job per adapter | did not exist | `CrawlerScheduler` with dynamic `CronJob` registration, `p-retry`-wrapped `runAdapter()` |
| `apps/api/src/crawler/crawler.scheduler.spec.ts` | Unit coverage for scheduling/retry | did not exist | 5 tests: disabled = zero jobs, one job per adapter, jobs stopped on destroy, retry-exhaustion swallows error after 5 calls, lock-not-acquired skips fetch |
| `apps/api/src/crawler/crawler.module.ts` | Nest module wiring | providers: `CRAWLER_ADAPTERS` factory only | adds `ScheduleModule.forRoot()` import + `AdvisoryLockService`/`CrawlerScheduler` providers |
| `apps/api/package.json` | Dependencies | no scheduling/retry libs | `@nestjs/schedule@^6.1.3`, `cron@^4.4.0` (CronJob, not re-exported by `@nestjs/schedule`), `p-retry@^4.6.2` (pinned to v4 for CJS interop) |

## 4. Code changes in detail

### 1. Advisory-lock connection lifetime — `apps/api/src/crawler/advisory-lock.service.ts`

**Before:** (file did not exist)

**After:**
```ts
@Injectable()
export class AdvisoryLockService {
  constructor(private readonly dataSource: DataSource) {}

  async withLock<T>(key: string, fn: () => Promise<T>): Promise<T | null> {
    const runner = this.dataSource.createQueryRunner();
    await runner.connect();
    let locked = false;
    try {
      const rows: Array<{ locked: boolean }> = await runner.query(
        "SELECT pg_try_advisory_lock(hashtext($1)) AS locked",
        [key],
      );
      locked = rows[0]?.locked === true;

      if (!locked) {
        this.logger.debug(`Lock "${key}" held elsewhere, skipping`);
        return null;
      }

      return await fn();
    } finally {
      if (locked) {
        await runner.query("SELECT pg_advisory_unlock(hashtext($1))", [key]);
      }
      await runner.release();
    }
  }
}
```

**What changed:** New service. `createQueryRunner()` + `.connect()` obtain one dedicated connection
that is used for both the `pg_try_advisory_lock` call and the matching `pg_advisory_unlock` call, and
`.release()` in `finally` always returns it to the pool.
**Why:** Postgres session-level advisory locks are scoped to the *connection*, not the app or a
transaction. If lock and unlock go through the pool's normal `Repository`/`DataSource.query()` calls
(which can each grab a different pooled connection), the unlock silently targets the wrong session
and no-ops — the lock leaks until that other connection's session eventually ends. This wasn't a bug
hit during the session; the phase file flagged it up front as "the single easiest thing to get wrong
in this phase," and the code was written correctly from the start using one runner for both calls.
**How it behaves now:** Two concurrent calls to `withLock("adapter-a", ...)` from different app
instances (or two overlapping ticks in the same instance) — the second gets `locked === false` from
`pg_try_advisory_lock` (non-blocking, returns immediately) and returns `null` without ever calling
`fn`. The first instance's lock is guaranteed released exactly once its `fn` settles, success or
throw.

### 2. Dynamic per-adapter cron registration with retry — `apps/api/src/crawler/crawler.scheduler.ts`

**Before:** (file did not exist)

**After:**
```ts
onModuleInit(): void {
  if (!this.config.enabled) {
    this.logger.log("Crawler disabled (CRAWLER_ENABLED != 'true'); no jobs registered");
    return;
  }

  for (const adapter of this.adapters) {
    const cronExpression = cronExpressionFor(this.config, adapter.category);
    const job = new CronJob(cronExpression, () => {
      void this.runAdapter(adapter);
    });
    this.schedulerRegistry.addCronJob(adapter.name, job);
    job.start();
    this.registeredJobNames.push(adapter.name);
  }
}

async runAdapter(adapter: ExternalDataSourceAdapter): Promise<void> {
  try {
    const result = await this.advisoryLockService.withLock(adapter.name, () =>
      pRetry(() => adapter.fetch(), {
        ...RETRY_OPTIONS,
        onFailedAttempt: (error) => {
          this.logger.warn(`[${adapter.name}] attempt ${error.attemptNumber} failed, ...`);
        },
      }),
    );
    if (result === null) return;
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    this.logger.error(`[${adapter.name}] tick failed after retries: ${message}`);
  }
}
```

**What changed:** New service. `CronJob` is imported from the `cron` package directly, not from
`@nestjs/schedule`; jobs are created and registered one per adapter inside a loop over
`CRAWLER_ADAPTERS`, rather than declared statically with `@Cron(expression)` decorators.
**Why:** `@Cron()` decorators bind a fixed expression to a fixed class method at compile time — they
can't express "N adapters, each with a cron expression looked up from config at runtime," which is
exactly this module's shape (adapters are injected via a DI array token that can vary by
environment/phase). `SchedulerRegistry.addCronJob` is the documented `@nestjs/schedule` escape hatch
for this case.
**How it behaves now:** With `CRAWLER_ENABLED=true` and N adapters registered, N independent cron
jobs run on their own schedules. Each tick acquires the adapter's lock, retries `fetch()` up to 4
times with exponential backoff (1s, 2s, 4s, 8s) on failure, and always resolves — never rejects —
even if every retry is exhausted, because the whole body is wrapped in try/catch. A disabled crawler
or empty adapter array registers zero jobs and the module is otherwise inert, same as phase 2.

### 3. Module wiring — `apps/api/src/crawler/crawler.module.ts`

**Before:**
```ts
@Module({
  imports: [TypeOrmModule.forFeature([CrawlerRunLog])],
  providers: [
    {
      provide: CRAWLER_ADAPTERS,
      useFactory: (): ExternalDataSourceAdapter[] => [],
    },
  ],
})
export class CrawlerModule {}
```

**After:**
```ts
@Module({
  imports: [TypeOrmModule.forFeature([CrawlerRunLog]), ScheduleModule.forRoot()],
  providers: [
    {
      provide: CRAWLER_ADAPTERS,
      useFactory: (): ExternalDataSourceAdapter[] => [],
    },
    AdvisoryLockService,
    CrawlerScheduler,
  ],
})
export class CrawlerModule {}
```

**What changed:** Added `ScheduleModule.forRoot()` to `imports`, added `AdvisoryLockService` and
`CrawlerScheduler` to `providers`.
**Why:** `ScheduleModule.forRoot()` is what makes `SchedulerRegistry` injectable; registering it here
(rather than in `AppModule`) keeps the cron subsystem's blast radius limited to the crawler feature —
if another future module also needs `@nestjs/schedule`, Nest allows `forRoot()` to be called from
multiple feature modules safely.
**How it behaves now:** `CrawlerScheduler` can now be constructor-injected with a working
`SchedulerRegistry`; previously this would have failed to resolve since nothing provided it.

### 4. Dependency additions — `apps/api/package.json`

**Before:** no `@nestjs/schedule`, `cron`, or `p-retry` entries.

**After:**
```json
"@nestjs/schedule": "^6.1.3",
...
"cron": "^4.4.0",
...
"p-retry": "^4.6.2",
```

**What changed:** Three new dependencies.
**Why:** `@nestjs/schedule` provides `SchedulerRegistry`/`ScheduleModule`. `cron` is needed directly
because `@nestjs/schedule` does not re-export `CronJob` (verified by reading its `dist/index.js` —
only enums, decorators, the module, and the registry are exported). `p-retry` is pinned to v4 rather
than the current major (v5+) because v5 ships as pure ESM, and this NestJS app compiles to CommonJS
via `nest build` / runs tests via `ts-jest` — a `require("p-retry")` smoke check confirmed v4's CJS
interop works with that toolchain.
**How it behaves now:** `pnpm --filter @dss/api build` and `pnpm --filter @dss/api test` both resolve
and run these imports without module-format errors.

## 5. How to find this again

- `grep -r "withLock" apps/api/src/crawler` — the advisory-lock wrapper and its one caller.
- `grep -r "addCronJob" apps/api/src/crawler` — dynamic per-adapter job registration.
- `grep -rn "pg_try_advisory_lock\|pg_advisory_unlock" apps/api/src` — the raw SQL, both call sites.
- `grep -rn "RETRY_OPTIONS\|p-retry" apps/api/src/crawler` — retry policy.
- File tree: `apps/api/src/crawler/{advisory-lock.service.ts,crawler.scheduler.ts}`.
- Plan: `plans/260828-0909-realtime-hydro-crawler/phase-03-scheduling-lock-and-retry.md`.

## 6. Concepts introduced

- **Postgres session-level advisory lock:** an application-level mutex (`pg_try_advisory_lock` /
  `pg_advisory_unlock`) keyed by an arbitrary integer, scoped to the *database connection* that
  acquired it — not to a transaction, not to the app process. Needed here to guarantee only one
  running instance of the API executes a given adapter's fetch at a time, without adding a schema
  table or manual stale-lock cleanup. The critical implication: lock and unlock must go through the
  *same* connection, or the unlock silently no-ops.
- **Dynamic vs. declarative NestJS scheduling:** `@Cron()` decorators bind a fixed schedule at class
  definition time; `SchedulerRegistry.addCronJob(name, job)` registers a `CronJob` instance built at
  runtime. Needed here because the set of adapters and their schedules come from a DI array + env
  config, not from a fixed set of decorated methods.
- **CommonJS/ESM interop constraint:** a pure-ESM npm package (`p-retry@5+`) cannot be `require()`d
  by a CJS-compiled NestJS build without extra bundler config. Needed to know this before picking a
  dependency version, not after a build failure.

## 7. Where it got stuck

**Symptom:** `import { CronJob } from "@nestjs/schedule"` did not resolve to the expected export.

**Cause:** `@nestjs/schedule` re-exports only its own decorators, `ScheduleModule`, and
`SchedulerRegistry` — it does not re-export the underlying `CronJob` class from the `cron` package it
wraps internally (confirmed by reading `@nestjs/schedule`'s `dist/index.js`).

**Fix:** Added `cron` as a direct dependency and imported `CronJob` from `cron` instead of
`@nestjs/schedule`.

---

**Symptom:** Considered adding a `CRAWLER_CONFIG` DI token (new `crawler.tokens.ts` file) so
`CrawlerConfig` could be constructor-injected into `CrawlerScheduler` the "Nest way."

**Cause / reasoning:** The phase file's file list only names four files to create (the two services
and their two spec files); a fifth token file wasn't in scope. It also risked a circular import
between `crawler.module.ts` (which would need to provide the token) and `crawler.scheduler.ts`
(which would consume it), since both already reference each other indirectly through the module.

**Fix (design decision, not a bug):** Kept `crawlerConfig()` as a plain function called directly in
`CrawlerScheduler`'s constructor, matching how it was already written in phase 2 (a standalone reader,
not wired into `ConfigService`). Still fully testable — `ts-jest` compiles named imports to a
`require()`-backed namespace object, so `jest.spyOn(crawlerConfigModule, "crawlerConfig")` works
without any DI token.

---

**Symptom:** The retry-exhaustion unit test (`crawler.scheduler.spec.ts`, "retries a failing fetch...")
timed out under Jest's default 5000ms per-test timeout.

**Cause:** The test uses real timers (not fake timers) against the real `p-retry` backoff — with
`minTimeout: 1000` and `factor: 2` across 4 retries, the waits alone total 1+2+4+8 = 15s, plus fetch
call and logging overhead. A first attempt at 15000ms was still occasionally too tight.

**Fix:** Set that test's own timeout to 20000ms (`it(..., 20000)`), rather than switching to fake
timers (which would require mocking `p-retry`'s internal delay mechanism — more invasive for one
test). Documented in Gotchas below since it makes this one test noticeably slow.

---

**Symptom:** `pnpm --filter @dss/api test` showed failures in `app.controller.spec.ts` and
`forecast/forecast.service.spec.ts` (same `StationRepository` DI resolution error already logged in
the phase 2 report).

**Cause (inferred, confirmed via evidence from this session):** Ran `git stash` to remove this
session's changes and re-ran the full suite against the clean tree — the same failures reproduced
identically with none of phase 3's code present, confirming they predate this session (already known
from phase 2's report as an orphaned `StationRepository` DI gap from phase 1).

**Fix:** None applied — left as-is and reported, per the phase's scope boundary; not phase 3's
responsibility to repair.

## 8. Verify

```bash
pnpm --filter @dss/api build
# expect: clean build, no TS errors

pnpm --filter @dss/api test -- crawler
# expect: 14 tests passed across advisory-lock.service.spec.ts, crawler.scheduler.spec.ts,
# crawler.config.spec.ts (retry-exhaustion test takes ~15-18s real time due to real p-retry backoff)

pnpm --filter @dss/api test
# expect: 61/68 tests passed; the 7 failures are all inside app.controller.spec.ts and
# forecast.service.spec.ts, confirmed pre-existing via git stash bisection (see section 7),
# unrelated to this session
```

**Not verified in this session:** the live-Postgres sanity check from the phase file's step 9 (two
sequential lock attempts on the same DB connection, confirming the second returns `false`) was not
run — no `.env` / live Postgres instance was available in this worktree. `AdvisoryLockService`'s
correctness rests on the mocked unit tests plus the code-review-level reasoning above, not a live-DB
confirmation. This is a real gap, not an oversight being glossed over.

## 9. Gotchas

- `AdvisoryLockService.withLock` opens a brand-new `QueryRunner` (i.e. a new pooled connection) on
  every call. Under high adapter counts or very frequent cron intervals this adds connection-pool
  pressure; if that ever becomes a problem, consider a longer-lived runner per adapter rather than
  per-tick, but that trades away the current "always fresh connection, no dangling state" simplicity.
- The retry-exhaustion test takes ~15-18s for real (real timers, no mocking of `p-retry`'s delay) —
  don't be alarmed if `crawler.scheduler.spec.ts` is visibly the slowest spec in the suite; that is
  expected, not a hang.
- `CRAWLER_ADAPTERS` is still an empty-array factory (phase 2) — `CrawlerScheduler.onModuleInit`
  currently registers zero jobs in any real environment until phase 4/5/7 add real adapter providers.
  Don't be surprised the crawler "does nothing" yet even with `CRAWLER_ENABLED=true`.
- The advisory-lock live-DB sanity check (phase file step 9) has never actually been run against
  Postgres in any worktree so far — if a future session finally has DB access, run it before trusting
  this pattern in production, don't assume the unit-test mocks are sufficient proof.
- `p-retry` is pinned to `^4.6.2` deliberately for CJS compatibility — do not bump it to v5+ without
  first confirming the build toolchain (webpack/esbuild config for `nest build`) can consume an
  ESM-only package, or the build will break.
