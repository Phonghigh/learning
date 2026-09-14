# devops-infra

Reports about deployment, service scripts, infrastructure, and environment/tooling failures.

| Date | Project | Title | What broke / what changed |
|---|---|---|---|
| 2026-08-17 | EWATER | [Tile server performance investigation ("/gis-map" slow)](SESSION-2026-08-17-EWATER-tile-server-perf-investigation.md) | User suspected slow raster viewing meant "move data to SSD"; measured disk cache and gunicorn worker count as candidate bottlenecks and disproved both with real timing + `docker stats` CPU data — root cause of the ~700-900ms/batch delay is still unidentified (suspected GDAL vsis3 HTTP overhead, unverified). Separately fixed a missing `maxzoom: 18` on the flood raster source. |
| 2026-08-16 | EWATER | [DEM tunnel silent crash + service extraction](SESSION-2026-08-16-EWATER-dem-tunnel-silent-crash.md) | Startup script died before writing any log after reboot; fixed with a PATH wait loop + try/catch, then moved the whole service out of the repo to `E:\Monitoring\FRIMS` |
