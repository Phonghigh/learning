# Session 2026-08-28 — Phase 1: crawler-ready schema, seed guards, drop `forecast_stations`

## 1. Requirement recap

Phase 1 ("DB migration + seed guards") of the "24/7 Real Hydro Data Crawler" plan
(`plans/260828-0909-realtime-hydro-crawler/phase-01-db-migration-and-seed-guards.md`): make the
Postgres/TypeORM schema safe for an automated crawler to repeatedly upsert data into, without
duplicating rows and without ever running the destructive fake-data seed scripts against a
production database.

## 2. How it was implemented + docs used

One migration, `20260829090000-InitCrawler.ts`, written as raw SQL (`up`/`down`) rather than
TypeORM's `synchronize` or auto-generated migration, so every step — including the two dedupe
`DELETE`s — is explicit and reviewable:

- `time_series_readings` gets a real `UNIQUE (station_id, metric_type, ts)` constraint (replacing a
  plain, non-unique index) so a crawler can `INSERT ... ON CONFLICT DO NOTHING` instead of
  duplicating readings on every poll. A dedupe `DELETE` (keep highest `id`) runs first since you
  cannot add a unique constraint over existing duplicate rows.
- `operation_events.occurred_at` loses its Postgres `now()` default and becomes settable directly
  (see bug 1 below for why `@CreateDateColumn` had to go). A nullable `external_id` column plus two
  **partial** unique indexes give the crawler two idempotency strategies depending on whether the
  upstream source exposes a stable id: `UQ_operation_events_external_id` when `external_id IS NOT
  NULL`, `UQ_operation_events_type_occurred` on `(event_type, occurred_at)` as a fallback when it
  isn't. Partial indexes (`WHERE ...`) were chosen over a single composite unique constraint because
  the two idempotency keys are mutually exclusive per row, not always-both-present.
- New `crawler_run_log` table for run observability (`source`, `started_at`, `finished_at`,
  `status` CHECK'd to `success|failure|partial`, `error_message`, `records_ingested`).
- `forecast_rainfall_values.station_id` is repointed from the standalone `forecast_stations` table
  (which only ever held 4 synthetic seed rows, see `20260826100000-ForecastExpansion.ts`) to the
  shared `stations` table, then `forecast_stations` is dropped outright. One station registry
  instead of two was the explicit goal — not a soft deprecation, a real `DROP TABLE` (see Gotchas).
  Since existing `forecast_rainfall_values` rows are 100% fake data pointing at ids that won't
  survive the drop, the table is `TRUNCATE`d first rather than attempting an impossible FK repoint
  of real data.
- `assertNotProduction(scriptName)` — a one-line guard thrown when `NODE_ENV === "production"` —
  added as the first statement of `seed-demo.ts` and `seed-fake-timeseries.ts`, both of which
  `TRUNCATE`/`DELETE` real tables. Documented in its own file comment as defense-in-depth, not the
  only control (prod DB credentials should never be present in a shell that runs seeds).

Docs used: none external — this was schema work grounded entirely in the existing codebase (prior
migration `20260826100000-ForecastExpansion.ts` for the exact `forecast_stations` `CREATE TABLE` +
seed data to copy into `down()`) and live queries against the dev Postgres container to verify
assumptions before writing SQL (see section 8).

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `apps/api/src/database/migrations/20260829090000-InitCrawler.ts` | Migration | did not exist | new: unique constraints, `crawler_run_log`, drops `forecast_stations` |
| `apps/api/src/database/seeds/assert-not-production.ts` | Guard helper | did not exist | new: throws if `NODE_ENV=production` |
| `apps/api/src/database/seeds/seed-demo.ts` | Seed script | ran unconditionally | calls `assertNotProduction("seed-demo")` first |
| `apps/api/src/database/seeds/seed-fake-timeseries.ts` | Seed script | ran unconditionally | calls `assertNotProduction("seed-fake-timeseries")` first |
| `apps/api/src/crawler/entities/crawler-run-log.entity.ts` | Entity | did not exist | new TypeORM entity for `crawler_run_log` |
| `apps/api/src/operation-log/entities/operation-event.entity.ts` | Entity | `occurredAt` via `@CreateDateColumn` (unsettable), no `externalId` | `occurredAt` plain `@Column({type:"timestamptz"})`, plus nullable `externalId` |
| `apps/api/src/monitoring/entities/time-series-reading.entity.ts` | Entity | `@Index(...)` non-unique | `@Unique(...)` |
| `apps/api/src/forecast/entities/forecast-rainfall-value.entity.ts` | Entity | `station: ForecastStation` | `station: Station` (shared entity) |
| `apps/api/src/forecast/entities/forecast-station.entity.ts` | Entity | existed | deleted |
| `apps/api/src/forecast/forecast.module.ts` / `forecast.service.ts` | Module/service | injected `ForecastStation` repo, unfiltered `find()` | injects shared `Station` repo, `find({where:{stationType:"rainfall"}})` |

## 4. Code changes in detail

### 1. Real uniqueness + dedupe on `time_series_readings` — `20260829090000-InitCrawler.ts`

**Before:** (no migration existed; the entity carried a plain non-unique index)
```ts
@Index("IDX_time_series_readings_station_metric_ts", ["station", "metricType", "ts"])
```

**After:**
```sql
DELETE FROM "time_series_readings" a
USING "time_series_readings" b
WHERE a."station_id" = b."station_id"
  AND a."metric_type" = b."metric_type"
  AND a."ts" = b."ts"
  AND a."id" < b."id";

ALTER TABLE "time_series_readings"
ADD CONSTRAINT "UQ_time_series_readings_station_metric_ts" UNIQUE ("station_id", "metric_type", "ts");

DROP INDEX "IDX_time_series_readings_station_metric_ts";
```
```ts
@Unique("UQ_time_series_readings_station_metric_ts", ["station", "metricType", "ts"])
```

**What changed:** the index became a constraint; a dedupe `DELETE` (keep highest `id`) runs first.
**Why:** `ON CONFLICT DO NOTHING` upserts — the crawler's core write pattern — require an actual
unique constraint or exclusion constraint on the conflict target; a plain index does not qualify.
**How it behaves now:** re-polling the same station/metric/timestamp is a silent no-op insert
instead of a duplicate row.

### 2. `operation_events` settable `occurred_at` + idempotency keys — same migration + entity

**Before (entity):**
```ts
import {
  Column,
  CreateDateColumn,
  Entity,
  JoinColumn,
  ManyToOne,
  PrimaryGeneratedColumn,
} from "typeorm";
...
@CreateDateColumn({ name: "occurred_at" })
occurredAt: Date;
```

**After (entity):**
```ts
import { Column, Entity, JoinColumn, ManyToOne, PrimaryGeneratedColumn } from "typeorm";
...
@Column({ name: "occurred_at", type: "timestamptz" })
occurredAt: Date;

/** External source's own event id, used for crawler upsert idempotency when available. */
@Column({ name: "external_id", type: "varchar", nullable: true })
externalId: string | null;
```

**After (migration SQL):**
```sql
ALTER TABLE "operation_events" ALTER COLUMN "occurred_at" DROP DEFAULT;
ALTER TABLE "operation_events" ADD COLUMN "external_id" varchar NULL;

DELETE FROM "operation_events" a
USING "operation_events" b
WHERE a."event_type" = b."event_type"
  AND a."occurred_at" = b."occurred_at"
  AND a."id" < b."id";

CREATE UNIQUE INDEX "UQ_operation_events_external_id" ON "operation_events" ("external_id")
  WHERE "external_id" IS NOT NULL;
CREATE UNIQUE INDEX "UQ_operation_events_type_occurred" ON "operation_events" ("event_type", "occurred_at")
  WHERE "external_id" IS NULL;
```

**What changed:** `@CreateDateColumn` (auto-stamped, un-overridable) replaced with a plain
`@Column`; added `external_id`; added a dedupe `DELETE` and two partial unique indexes.
**Why:** `@CreateDateColumn` silently ignores any value the caller supplies at insert time — a
crawler needs to store the *real* upstream event timestamp, not the moment it happened to be
ingested (see Bug 1, section 7).
**How it behaves now:** the crawler can set `occurredAt` to the source's real timestamp; a second
insert for the same external id (or, lacking one, the same type+timestamp) is rejected/ignored
rather than duplicated.

### 3. `crawler_run_log` table + entity — new files

**Before:** did not exist.
**After (SQL, migration):**
```sql
CREATE TABLE "crawler_run_log" (
  "id" uuid NOT NULL DEFAULT gen_random_uuid(),
  "source" varchar NOT NULL,
  "started_at" timestamptz NOT NULL,
  "finished_at" timestamptz NULL,
  "status" varchar NOT NULL,
  "error_message" text NULL,
  "records_ingested" integer NOT NULL DEFAULT 0,
  CONSTRAINT "PK_crawler_run_log" PRIMARY KEY ("id"),
  CONSTRAINT "CHK_crawler_run_log_status" CHECK ("status" IN ('success', 'failure', 'partial'))
);
CREATE INDEX "IDX_crawler_run_log_source_started" ON "crawler_run_log" ("source", "started_at" DESC);
```
**What changed:** brand-new table + `crawler-run-log.entity.ts` TypeORM entity.
**Why:** the crawler needs an audit trail of every run (success/failure/partial, how many records)
for operational visibility once it runs unattended 24/7.
**How it behaves now:** each crawler execution can write a start row, then update it with
`finished_at`/`status`/`records_ingested`, queryable by source and recency.

### 4. Drop `forecast_stations`, repoint FK to shared `stations` — migration + 4 files

**Before (entity):**
```ts
import { ForecastStation } from "./forecast-station.entity";
...
@ManyToOne(() => ForecastStation, { onDelete: "CASCADE" })
@JoinColumn({ name: "station_id" })
station: ForecastStation;
```
**After (entity):**
```ts
import { Station } from "../../monitoring/entities/station.entity";
...
@ManyToOne(() => Station, { onDelete: "CASCADE" })
@JoinColumn({ name: "station_id" })
station: Station;
```
**After (migration SQL):**
```sql
TRUNCATE TABLE "forecast_rainfall_values";
ALTER TABLE "forecast_rainfall_values" DROP CONSTRAINT "forecast_rainfall_values_station_id_fkey";
ALTER TABLE "forecast_rainfall_values"
  ADD CONSTRAINT "FK_forecast_rainfall_values_station" FOREIGN KEY ("station_id")
    REFERENCES "stations"("id") ON DELETE CASCADE;
DROP TABLE "forecast_stations";
```
**After (forecast.service.ts):**
```ts
// before: const rows = await this.stationRepo.find({ order: { name: "ASC" } });
const rows = await this.stationRepo.find({ where: { stationType: "rainfall" }, order: { name: "ASC" } });
```
**What changed:** `ForecastRainfallValue.station` now points at the shared `Station` entity;
`forecast.module.ts`/`forecast.service.ts` inject `Station`'s repo instead of the deleted
`ForecastStation` repo; `getStations()` gained a `where: { stationType: "rainfall" }` filter to
reproduce the same result set the dedicated table gave for free; `forecast-station.entity.ts`
deleted; `forecast_stations` table dropped.
**Why:** the crawler should write to one station registry, not maintain two overlapping tables
(`stations` and `forecast_stations`) that could drift out of sync.
**How it behaves now:** rainfall stations shown on the Forecast Overview dashboard come from
`stations` filtered by `station_type = 'rainfall'`; any code still importing the old
`ForecastStation` entity fails to compile (see Gotchas).

### 5. `assertNotProduction` guard — new file + 2 call sites

**Before:** `seed-demo.ts` / `seed-fake-timeseries.ts` `run()` started directly with
`await dataSource.initialize()`.
**After:**
```ts
export function assertNotProduction(scriptName: string): void {
  if (process.env.NODE_ENV === "production") {
    throw new Error(
      `${scriptName} refused to run: NODE_ENV is "production". This script writes/truncates fake demo data and must never touch a production database.`,
    );
  }
}
```
```ts
async function run(): Promise<void> {
  assertNotProduction("seed-demo");
  await dataSource.initialize();
  ...
```
**What changed:** one new file, one new call as the first line of each seed script's `run()`.
**Why:** both scripts `TRUNCATE`/`DELETE` real tables (`forecast_values`, `stations`,
`operation_events`, `alerts`, etc.) to reseed fake demo data — an accidental run against production
would be destructive and irreversible.
**How it behaves now:** either script throws immediately and does nothing further if
`NODE_ENV=production` is set in the environment it runs in.

## 5. How to find this again

- `InitCrawler20260829090000` — migration class name / file
  `apps/api/src/database/migrations/20260829090000-InitCrawler.ts`
- `assertNotProduction` — grep across `apps/api/src/database/seeds/`
- `UQ_time_series_readings_station_metric_ts`, `UQ_operation_events_external_id`,
  `UQ_operation_events_type_occurred` — constraint/index names, grep in migrations or `psql \d`
- `crawler_run_log` — table name, entity at `apps/api/src/crawler/entities/crawler-run-log.entity.ts`
- `getStations()` in `apps/api/src/forecast/forecast.service.ts` — now filters by `stationType`

## 6. Concepts introduced

- **Partial unique index** (Postgres): a `CREATE UNIQUE INDEX ... WHERE <condition>` that only
  enforces uniqueness over rows matching the condition. Needed here because `operation_events` has
  two mutually-exclusive idempotency strategies (by `external_id` when present, by
  `(event_type, occurred_at)` otherwise) that cannot both be a single always-active constraint.
- **`ON CONFLICT DO NOTHING` (upsert)**: a Postgres `INSERT` clause that silently skips the row
  instead of erroring when it would violate a unique/exclusion constraint. This is the entire reason
  the migration adds real unique constraints — without one, there is no conflict target for the
  crawler's inserts to target.
- **`@CreateDateColumn` vs plain `@Column({type:"timestamptz"})`** (TypeORM): `CreateDateColumn`
  auto-stamps the current time on insert and ignores any explicitly assigned value — a trap for any
  column meant to hold an externally-sourced timestamp rather than "when this row was created here."

## 7. Where it got stuck

**Bug 1 — `DataTypeNotSupportedError: Data type "Object" ... is not supported by "postgres"`**
Symptom: `pnpm migration:run` threw this error at `DataSource.initialize()`, before any SQL ran.
Cause: `@Column({ name: "external_id", nullable: true }) externalId: string | null;` had no explicit
`type:` option. TypeORM's reflection (`design:emit-decorator-metadata`) on a `string | null` union
type resolves to the generic `Object` type at the TS-metadata level, which the postgres driver
rejects as a column type. Confirmed by checking that the existing `Station.code` entity property —
also `string | null` — has `@Column({ type: "varchar", nullable: true })` with an explicit type,
i.e. the codebase already had the correct pattern; this session initially omitted it.
Fix: added `type: "varchar"` explicitly: `@Column({ name: "external_id", type: "varchar", nullable: true })`.

**Bug 2 — `23505 duplicate key value violates unique constraint` on `UQ_operation_events_type_occurred`**
Symptom: after fixing bug 1, `pnpm migration:run` against the live dev Postgres container failed
partway through with: `could not create unique index ... Key (event_type, occurred_at)=(gate_operation,
2026-08-25 08:14:29.685+00) is duplicated.`
Cause (inferred from reading `seed-demo.ts`, corroborated by the failing values in the error message
matching seeded data): `seed-demo.ts` inserts 3 `gate_operation` rows in a loop that all reuse one
`Date` computed once outside the loop (`now.toISOString()`), so they share an identical
`occurred_at`. The dev DB had this pre-existing seed data with genuine duplicates on
`(event_type, occurred_at)`, which the new partial unique index rejected. The phase plan file only
scoped a dedupe step for `time_series_readings`, not `operation_events` — this gap was found live,
not anticipated in planning.
Fix: added a second dedupe `DELETE FROM "operation_events" ... WHERE a.id < b.id` on
`(event_type, occurred_at)`, placed after adding `external_id` and before creating the two partial
indexes — a deviation from the phase plan added reactively once the live-DB failure surfaced it.

## 8. Verify

- `docker exec dss-song-ray-postgres psql -U dss -d dss_song_ray -c "\d forecast_rainfall_values"` —
  confirmed the real FK constraint name (`forecast_rainfall_values_station_id_fkey`) before writing
  the `DROP CONSTRAINT` statement, rather than guessing TypeORM's default naming.
- `docker exec dss-song-ray-postgres psql -U dss -d dss_song_ray -c "SELECT station_id, metric_type,
  ts, COUNT(*) FROM time_series_readings GROUP BY 1,2,3 HAVING COUNT(*) > 1 LIMIT 20;"` — returned 0
  rows, confirming the `time_series_readings` dedupe `DELETE` is a safe no-op in this environment.
- From `apps/api`: `pnpm migration:run` → `pnpm migration:revert` → `pnpm migration:run` — full cycle
  succeeded against the live dev DB, proving `down()` is a correct, working reversal (including
  recreating `forecast_stations` with its original seed data).
- `pnpm --filter @dss/api build` — clean Nest build both before and after the change.
- `pnpm test -- operation-log time-series` — 2 suites, 6 tests, all passing.
- Note: `forecast.service.spec.ts` has 7 pre-existing failures unrelated to this session — confirmed
  via `git stash` that they fail identically on pre-session commit `af35c85` (missing
  `StationRepository`/`ForecastStationRepository` providers in its NestJS `TestingModule` setup, a
  pre-existing gap in the test file, not something this session introduced or was asked to fix).

## 9. Gotchas

- Any new nullable TypeORM `@Column` on a union type like `string | null` **must** carry an explicit
  `type:` option, or migration generation/run fails with a cryptic `Data type "Object" is not
  supported` error at `DataSource.initialize()` — easy to hit again in this codebase; follow the
  existing `Station.code` pattern.
- When adding **any** new unique constraint/index via a raw-SQL migration, run a duplicate-check
  `SELECT ... GROUP BY ... HAVING COUNT(*) > 1` against the live dev DB first, even for tables the
  plan doesn't flag as risky — seed data can silently violate a new uniqueness rule the plan never
  anticipated (as happened with `operation_events` here).
- `forecast_stations` is a hard `DROP TABLE`, not a soft deprecation. Any branch/PR still referencing
  the deleted `ForecastStation` entity will fail to compile and must be redirected to the shared
  `Station` entity filtered by `stationType: "rainfall"`.
- `forecast_rainfall_values` was `TRUNCATE`d as part of this migration (its data was 100% synthetic
  seed data) — `down()` restores an empty table, not the original fake rows; this is intentional, not
  a bug in the reversal.
