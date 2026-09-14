# Session 2026-08-28 — Realtime hydro crawler, Phase 2: module skeleton and adapter seam

## 1. Requirement recap

Phase 2 of `plans/260828-0909-realtime-hydro-crawler/phase-02-crawler-module-skeleton.md`: build only
the skeleton of `apps/api/src/crawler/` — no scheduling (phase 3), no real adapters (phases 4/5/7).
The concrete ask was to design an extensible seam so that unknown-shape future adapters (station
telemetry scraping, operation-event polling, rainfall forecast fetch) can be plugged in later without
reshaping the module.

## 2. How it was implemented + docs used

Four design decisions, each deliberately minimal (YAGNI):

- **`RawReading` discriminated union on `category`** instead of one loose interface with optional
  fields. This forces `IngestService` (phase 4) to write an exhaustive `switch` — the compiler flags
  missing branches when phase 7 adds a new category (`station_rainfall_forecast`).
- **`CRAWLER_ADAPTERS` as a DI token resolving to an array**, not a hardcoded list of provider
  classes. Phases 3-5 add adapters by registering more providers against this token; the module,
  scheduler, and ingest layer never change shape.
- **`fetch(): Promise<RawReading[]>`, flat, not paginated/cursor-based.** Explicit rejection of
  building `fetchSince(since: Date)` now — documented in the phase file's risk table as "add later
  if a source needs incremental state, don't reshape the interface speculatively."
- **`crawlerConfig()` as the single `process.env` read point**, with a strict `CRAWLER_ENABLED ===
  "true"` check (not truthy) so any unset/misspelled value defaults to *disabled* — a safe default
  for local dev/CI, and a 5-field whitespace-split validation on each cron string that throws at
  boot rather than failing silently at schedule time.

No existing code needed reshaping; `CrawlerRunLog` entity was already created in phase 1
(commit `636ef04`) and is just wired via `TypeOrmModule.forFeature` here.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/api/src/crawler/adapters/raw-reading.types.ts` | Discriminated union of raw payload shapes an adapter can emit | did not exist | `RawReading` union on `category`: telemetry / operation_event / rainfall_forecast |
| `apps/api/src/crawler/adapters/external-data-source.adapter.ts` | Adapter contract + DI token | did not exist | `ExternalDataSourceAdapter` interface, `CRAWLER_ADAPTERS` Symbol token, `CrawlerCategory` type |
| `apps/api/src/crawler/crawler.config.ts` | Single `process.env` read point for crawler settings | did not exist | `crawlerConfig()`: enabled flag (strict `"true"` check) + 3 validated cron strings |
| `apps/api/src/crawler/crawler.config.spec.ts` | Unit coverage for config parsing | did not exist | 5 tests: defaults, strict-enable, env override, malformed cron, empty cron |
| `apps/api/src/crawler/crawler.module.ts` | Nest module wiring | did not exist | Registers `CrawlerRunLog` via TypeORM + `CRAWLER_ADAPTERS` factory returning `[]` |
| `apps/api/src/app.module.ts` | Root module imports | `FloodModule` was last import | `CrawlerModule` imported and registered after `FloodModule` |
| `apps/api/.env.example` | Documented env vars | no crawler vars | 4 new vars: `CRAWLER_ENABLED`, `CRAWLER_RAINFALL_FORECAST_CRON`, `CRAWLER_TELEMETRY_CRON`, `CRAWLER_OPERATION_EVENT_CRON` |

## 4. Code changes in detail

### 1. Discriminated union for adapter output — `apps/api/src/crawler/adapters/raw-reading.types.ts`

**Before:** (file did not exist)

**After:**
```ts
export interface RawTelemetryReading {
  stationCode: string;
  metricType: "rainfall_mm" | "water_level_m" | "discharge_m3s";
  ts: Date;
  value: number;
}

export interface RawOperationEvent {
  externalId: string | null;
  eventType: string;
  occurredAt: Date;
  description: string | null;
  metadata: Record<string, unknown> | null;
}

export interface RawRainfallForecastPoint {
  stationCode: string;
  validAt: Date;
  valueMm: number;
}

export type RawReading =
  | { category: "telemetry"; data: RawTelemetryReading }
  | { category: "operation_event"; data: RawOperationEvent }
  | { category: "rainfall_forecast"; data: RawRainfallForecastPoint };
```

**What changed:** New file defining three payload shapes plus a tagged union over them.
**Why:** A single loose "raw reading" interface with optional fields (`value?`, `eventType?`, ...)
would let a consumer silently skip handling a category — TypeScript wouldn't complain. Tagging on
`category` turns "did you handle rainfall_forecast?" into a compile error if you don't.
**How it behaves now:** When `IngestService` is built in phase 4, a `switch (reading.category)`
without a `default` gets a TS2366 (not all code paths return a value) or similar exhaustiveness
error the moment a new category is added anywhere in the union — the compiler does the review, not
a human remembering to grep for every switch on `category`.

### 2. Adapter contract and DI token — `apps/api/src/crawler/adapters/external-data-source.adapter.ts`

**Before:** (file did not exist)

**After:**
```ts
export type CrawlerCategory = "telemetry" | "operation_event" | "rainfall_forecast";

export interface ExternalDataSourceAdapter {
  readonly name: string;
  readonly category: CrawlerCategory;
  fetch(): Promise<RawReading[]>;
}

export const CRAWLER_ADAPTERS = Symbol("CRAWLER_ADAPTERS");
```

**What changed:** New interface + a `Symbol`-based DI token (not a string token, to avoid collision
with any other module's token of the same name).
**Why:** Nest's constructor injection needs *something* to key providers by when the type is an
interface (interfaces don't exist at runtime, so `@Inject(ExternalDataSourceAdapter)` isn't valid).
**How it behaves now:** `crawler.module.ts` currently provides `CRAWLER_ADAPTERS` as an empty-array
factory. Phase 3+ can add `{ provide: CRAWLER_ADAPTERS, useClass: TelemetryAdapter, multi: true }`
style registration (or an array factory that composes multiple adapter classes) without any other
file in the module changing.

### 3. Env-driven config with boot-time cron validation — `apps/api/src/crawler/crawler.config.ts`

**Before:** (file did not exist)

**After:**
```ts
function assertValidCron(value: string, envVarName: string): string {
  const trimmed = value.trim();
  if (trimmed.length === 0 || trimmed.split(/\s+/).length !== 5) {
    throw new Error(`Invalid cron expression for ${envVarName}: "${value}"`);
  }
  return trimmed;
}

export function crawlerConfig(): CrawlerConfig {
  return {
    enabled: process.env.CRAWLER_ENABLED === "true",
    cron: {
      rainfallForecast: assertValidCron(
        process.env.CRAWLER_RAINFALL_FORECAST_CRON ?? DEFAULT_RAINFALL_FORECAST_CRON,
        "CRAWLER_RAINFALL_FORECAST_CRON",
      ),
      // ... telemetry, operationEvent follow the same pattern
    },
  };
}
```

**What changed:** New function; `enabled` uses strict equality against the literal string `"true"`,
not `Boolean(process.env.CRAWLER_ENABLED)` (which would be `true` for `"false"`, `"0"`, any non-empty
string — a classic string-to-boolean footgun) and not `=== "1" || === "true"` (extra surface to keep
in sync). Each cron string gets a 5-field whitespace-split check before use.
**Why:** A misconfigured cron string should fail loudly at process boot, not silently no-op or crash
later inside a scheduled job handler where the failure is harder to trace back to the env var.
**How it behaves now:** `CRAWLER_ENABLED=false`, unset, or any typo (`"True"`, `"yes"`) all resolve
to `enabled: false` — the crawler defaults to off unless the exact string `"true"` is set. A cron
string like `"not-a-cron"` throws `Invalid cron expression for CRAWLER_TELEMETRY_CRON: "not-a-cron"`
at the point `crawlerConfig()` is called (module init), not when `@nestjs/schedule` first tries to
parse it during a job run.

### 4. Module wiring — `apps/api/src/crawler/crawler.module.ts`

**Before:** (file did not exist)

**After:**
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

**What changed:** New module registering the phase-1 `CrawlerRunLog` entity for repository injection
and providing `CRAWLER_ADAPTERS` as an empty-array factory.
**Why:** Keeps the module valid and importable today (Nest requires the token to resolve to
*something*) while leaving the actual adapter list for phases 3-5 to populate.
**How it behaves now:** `CrawlerModule` compiles and boots cleanly with zero adapters registered — no
scheduling runs, no HTTP calls happen, `AppModule` starts up exactly as before this commit except for
the added (currently inert) module.

### 5. Registration in the root module — `apps/api/src/app.module.ts`

**Before:**
```ts
import { FloodModule } from "./flood/flood.module";

@Module({
  imports: [
    // ...
    FloodModule,
  ],
```

**After:**
```ts
import { FloodModule } from "./flood/flood.module";
import { CrawlerModule } from "./crawler/crawler.module";

@Module({
  imports: [
    // ...
    FloodModule,
    CrawlerModule,
  ],
```

**What changed:** One import line, one entry in the `imports` array.
**Why:** Without this, `CrawlerModule` is dead code — Nest only instantiates modules reachable from
the root module graph.
**How it behaves now:** `CrawlerRunLog`'s repository and `CRAWLER_ADAPTERS` become available for
injection anywhere `CrawlerModule` is imported (or exported from, once it needs to be — not yet).

## 5. How to find this again

- `grep -r "CRAWLER_ADAPTERS" apps/api/src` — the DI token and its one current provider.
- `grep -r "RawReading" apps/api/src/crawler` — the discriminated union and where it will be consumed.
- `grep -r "crawlerConfig" apps/api/src` — the single env-read point.
- File tree: `apps/api/src/crawler/{adapters/,crawler.module.ts,crawler.config.ts}`.
- Plan: `plans/260828-0909-realtime-hydro-crawler/phase-02-crawler-module-skeleton.md`.

## 6. Concepts introduced

- **Discriminated union (TypeScript):** a union type where each member shares a common literal field
  (`category` here) that TypeScript uses to narrow which shape you're dealing with inside a
  conditional or `switch`. Needed here so future consumers get compile-time exhaustiveness checks
  when a new reading category is added, instead of relying on someone remembering to update every
  switch statement.
- **DI token via `Symbol`:** in NestJS, when you want to inject something that's only an interface at
  compile time (interfaces vanish at runtime), you need a runtime-existing key — a `Symbol` (or
  string) — to register and resolve providers against. Needed here because `ExternalDataSourceAdapter`
  is an interface, and the module wants to support *multiple* adapters resolving to one array.
- **Factory provider:** a Nest provider defined with `useFactory` instead of `useClass`/`useValue`,
  letting the returned value be computed (here, an array assembled from future registrations) rather
  than being a single class instance.

## 7. Where it got stuck

**Symptom:** After implementing phase 2 and running the full API test suite, 7 tests failed in
`forecast.service.spec.ts`.

**Initial concern:** Did the phase 2 crawler module changes (module registration in `app.module.ts`,
new files) break something in an unrelated forecast service test?

**Investigation:** Ran `git stash` to remove all phase 2 changes, then re-ran `forecast.service.spec`
in isolation against the stashed (phase-1-only) tree — the same 7 failures reproduced with none of
the crawler files present. Restored the changes with `git stash pop`.

**Conclusion (inferred, but backed by the stash test above):** The failures are pre-existing and
orphaned from phase 1 (commit `636ef04`), which repointed `forecast_rainfall_values.station_id` from
the deleted `ForecastStation` entity to the shared `stations` table but didn't update
`forecast.service.spec.ts`'s `TestingModule` providers to include `StationRepository` — so the spec's
DI container can't resolve a dependency the service now needs. This is out of phase 2's scope (a
module-skeleton task shouldn't be repairing an unrelated spec's DI wiring) and was left as-is,
documented rather than silently fixed, per the phase file's stated scope boundary.

**False leads ruled out:** Considered that `CrawlerModule`'s `TypeOrmModule.forFeature([CrawlerRunLog])`
registration in `app.module.ts` might be interfering with TypeORM's global connection/entity
metadata and somehow affecting an unrelated module's tests. Ruled out by the stash test — the
failures existed identically with `CrawlerModule` entirely absent from the tree.

## 8. Verify

```bash
pnpm --filter @dss/api build
# expect: clean build, no TS errors

pnpm --filter @dss/api test -- crawler.config
# expect: 5 passed, 5 total (crawler.config.spec.ts)

pnpm --filter @dss/api test
# expect: same 7 pre-existing forecast.service.spec.ts failures (unrelated to this session,
# confirmed via git stash bisection above), all other suites (52 tests) passing
```

## 9. Gotchas

- `CRAWLER_ADAPTERS` currently resolves to a hardcoded empty-array factory. When phase 3/4/5 adds
  real adapters, remember to replace the factory body (not add a second provider for the same
  token — Nest providers for the same token overwrite, they don't merge).
- `crawlerConfig()` is a plain function, not wired into Nest's `ConfigModule`/`@nestjs/config` yet.
  If a future phase introduces `ConfigModule.forRoot()` validation schemas, decide whether
  `crawler.config.ts` gets absorbed into that or stays a standalone reader — don't end up with two
  independent env-parsing paths for the same variables.
- The `CRAWLER_ENABLED === "true"` strict check means `CRAWLER_ENABLED=True` or `CRAWLER_ENABLED=1`
  silently do nothing (crawler stays off) rather than erroring — intentional as a safe default, but
  worth remembering when debugging "why isn't the crawler running" in an actual deployed env file.
- The pre-existing `forecast.service.spec.ts` DI gap (missing `StationRepository` provider) is still
  unfixed. Anyone touching `forecast.service.spec.ts` next should expect those 7 failures and know
  they predate this session.
