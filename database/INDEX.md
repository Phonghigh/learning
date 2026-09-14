# database

Schema design, migrations, constraints, seed data, and query-layer decisions.

| Date | Project | Title | What broke / what changed |
|---|---|---|---|
| 2026-08-28 | DSS_Song_Ray | [Phase 1: crawler-ready schema, seed guards, drop `forecast_stations`](SESSION-2026-08-28-DSS_Song_Ray-realtime-hydro-crawler-phase1-db-migration.md) | New `InitCrawler` migration adds a real `UNIQUE` constraint on `time_series_readings(station_id, metric_type, ts)` for crawler `ON CONFLICT` upserts, converts `operation_events.occurred_at` from `@CreateDateColumn` to a settable column plus two partial unique indexes for idempotency, adds a `crawler_run_log` audit table, and drops `forecast_stations` in favor of the shared `stations` table. Added an `assertNotProduction` guard to both destructive fake-data seed scripts. Hit two bugs live: a `string \| null` `@Column` without explicit `type: "varchar"` threw `DataTypeNotSupportedError` (TypeORM reflects unioned types as `Object`), and pre-existing demo-seeded `operation_events` rows sharing one timestamp violated the new partial unique index — fixed with a second dedupe `DELETE` not scoped in the original plan. |
