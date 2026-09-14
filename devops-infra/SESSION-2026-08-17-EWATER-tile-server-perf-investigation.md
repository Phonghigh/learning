# SESSION-2026-08-17 — Tile server performance investigation ("/gis-map" chậm)

**Date:** 2026-08-17

---

## 1. Requirement recap

User's own framing (Vietnamese): "đang gặp vấn đề trong chạy mô phỏng, nó chậm quá, không ổn, hãy
phân tích đánh giá xem có thể cải thiện tốc độ như nào, có thể chuyển qua dùng SSD ổ C:\"

Clarified via follow-up question: "mô phỏng chậm" meant viewing/scrubbing through the flood raster
layer on the `/gis-map` page — not running an ingest job and not a hydraulic engine (the repo has no
live simulation engine; `simulation.json` is a pre-generated 52-step scenario read from disk). The
underlying ask: is the raster on a slow disk (D:, HDD), and would moving it to C: (SSD) fix the lag?

## 2. How it was implemented + docs used

Diagnostic session, not a feature build. Approach: measure before guessing, at every layer (disk
cache state, worker count, CPU/RAM under load) rather than accepting the first plausible-looking
number as the cause.

- `Get-PhysicalDisk` / `Get-Volume` (PowerShell, read-only) to confirm which drive is SSD vs HDD and
  how much free space each has.
- Direct `curl` calls to the already-running titiler container (`127.0.0.1:8090`) to reproduce a
  realistic MapLibre burst: 24 tiles at z17 in a 5x5 grid around the AOI centre, fired in parallel
  (`&` + `wait` in bash), timed with `date +%s%N`.
- `docker stats --no-stream` run concurrently with a burst to see actual CPU/RAM consumption of the
  `titiler` and `minio` containers during the slow window — this is what ultimately ruled out the
  worker-count theory.
- `curl .../cog/info?url=...` to confirm the flood raster grid resolution matches the DEM grid
  (13709x8382 px, ~0.5m/pixel) — this is what justified the `maxzoom: 18` fix, reusing the exact
  reasoning already applied to the DEM layer.

Two theories were tested and both were rejected by measurement (see §7). No production code change
was made based on either theory — `WEB_CONCURRENCY` was reverted to its original value. The only
code change kept is the `maxzoom: 18` fix, which is unrelated to the ~700-900ms/batch delay under
investigation and does not fully explain it.

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `services/tile-server/docker-compose.yml` | titiler gunicorn worker count | `WEB_CONCURRENCY=8`, comment claimed 8 workers roughly halved batch latency vs 1 worker | still `WEB_CONCURRENCY=8` (unchanged value), comment now also records that 8→16 was tried and reverted — no measured benefit, and `docker stats` showed CPU was never the bottleneck |
| `web/src/components/map/useMapContextLayers.ts` | MapLibre raster source config for the flood layer | `addSource("flood-raster", { type: "raster", tiles: [tileUrl], tileSize: 256 })` (both `onLoad` init and the timeline-step-change effect) — no `maxzoom`, unlike the DEM layer which already had `maxzoom: 18` | both call sites now pass `maxzoom: 18`, matching the DEM layer's existing behavior, because the flood raster shares the same ~0.5m/pixel AOI grid |

## 4. Code changes in detail

### 1. Revert `WEB_CONCURRENCY` after disproving the worker-bottleneck theory — `services/tile-server/docker-compose.yml`

**Before:**
```yaml
      # thật (curl 30 tile song song, xem báo cáo phân tích hiệu năng): 1
      # worker mất 2.1s, 8 worker mất 1.0s cho CÙNG batch. MapLibre bắn hàng
      # chục tile request song song mỗi lần pan/zoom -1 worker khiến chúng
      # xếp hàng lần lượt dù host có 12 CPU rảnh gần như 100%. "8" là số CPU
      # host đo được lúc test (Docker Desktop báo 12 CPU); không đặt bằng
      # đúng số CPU host để chừa margin cho request khác + tránh quá tải
      # nếu host đổi (deploy production).
      - WEB_CONCURRENCY=8
```

**After:**
```yaml
      # thật (curl 30 tile song song, xem báo cáo phân tích hiệu năng): 1
      # worker mất 2.1s, 8 worker mất 1.0s cho CÙNG batch. MapLibre bắn hàng
      # chục tile request song song mỗi lần pan/zoom -1 worker khiến chúng
      # xếp hàng lần lượt dù host có 12 CPU rảnh gần như 100%.
      #
      # ĐÃ THỬ 8 -> 16 (2026-08-17, điều tra raster ngập chậm trên
      # /gis-map) và ĐÃ REVERT: không cải thiện gì (24 tile song song z17
      # vẫn ~660-750ms ở cả 8 và 16 worker), trong khi CPU cả titiler
      # (10%) lẫn MinIO (0.3%) đều thấp lúc burst -> không phải nghẽn
      # worker/CPU. Cache HĐH nguội/ấm cũng không đổi kết quả -> không phải
      # nghẽn đĩa. Nghi ngờ hiện tại: chi phí round-trip HTTP range-request
      # của GDAL vsis3 (nhiều request nhỏ/tile qua MinIO) là chi phí cố
      # định không phụ thuộc CPU/đĩa -CHƯA verify, cần đo riêng nếu muốn
      # tối ưu tiếp (không đoán thêm, xem learn-log 2026-08-17). Giữ 8 vì
      # 16 tốn gấp đôi RAM (2.5GB vs 1.5GB) mà không có lợi ích đo được.
      - WEB_CONCURRENCY=8
```

**What changed:** the numeric value ended up the same (`8`), but a `docker compose up -d` cycle was
done twice (8→16, then 16→8) in between — this is a real config round-trip, not a no-op. The comment
was rewritten to record the new negative result.
**Why:** without this, a future session (or the same user in six months) would re-read the old
comment, see it only justifies "8 vs 1", and try bumping the worker count again as the first
optimization idea — wasting the exact 20 minutes this session already spent disproving it.
**How it behaves now:** functionally unchanged (still 8 gunicorn workers). The container currently
uses ~1.5GB RAM instead of the ~2.5GB it briefly used during the 16-worker test.

### 2. Add `maxzoom: 18` to the flood raster source (both call sites) — `web/src/components/map/useMapContextLayers.ts`

**Before:**
```ts
map.addSource("flood-raster", { type: "raster", tiles: [tileUrl], tileSize: 256 });
```
*(identical in both the `onLoad` init effect around line 105 and the timeline-step-change effect
around line 366; the DEM layer in the same file already had `maxzoom: 18` with a comment explaining
why, dated earlier than this session.)*

**After:**
```ts
// `maxzoom: 18` -cùng lý do DEM (`demTileUrlTemplate`): raster ngập
// dùng CHUNG lưới AOI ~0.5m/pixel (verify qua `/cog/info`,
// 13709x8382px), không có chi tiết mới quá z18-19. Thiếu giới hạn
// này khiến MapLibre xin tile ở zoom sâu hơn cần thiết khi zoom
// gần -tốn request vô ích (2026-08-17, điều tra "/gis-map chậm").
map.addSource("flood-raster", { type: "raster", tiles: [tileUrl], tileSize: 256, maxzoom: 18 });
```

**What changed:** added the `maxzoom: 18` property to the `addSource` options object at both call
sites; added an explanatory comment above the first one.
**Why:** the flood raster COG has the same ~0.5m/pixel resolution as the DEM COG (confirmed via
`curl .../cog/info`, 13709x8382 px). Without `maxzoom`, MapLibre keeps requesting deeper zoom tiles
past the point where the source has real detail, generating tile requests that return upsampled/
interpolated pixels for no visual gain.
**How it behaves now:** past z18, MapLibre reuses the z18 tiles instead of firing new requests —
fewer wasted requests when a user zooms in close on the flood layer. This is a real, independent
improvement, but it does not explain the ~700-900ms/batch delay measured at z17 (see §7) — that
delay happens at a zoom level below where this fix applies.

## 5. How to find this again

- `grep -rn "WEB_CONCURRENCY" services/tile-server/`
- `grep -rn "flood-raster" web/src/components/map/useMapContextLayers.ts`
- `grep -rn "maxzoom" web/src/components/map/useMapContextLayers.ts web/src/lib/demLayer.ts`
- Route: `/gis-map`. Container names: `titiler`, `minio` (see `services/tile-server/docker-compose.yml`).

## 6. Concepts introduced

### COG (Cloud-Optimized GeoTIFF)
- **Plain definition:** a GeoTIFF file organized so a server can read only the small byte-range
  needed for one tile, instead of downloading the whole file.
- **Why it shows up here:** the flood raster and DEM are both served this way from MinIO; the theory
  under investigation (unverified) is that the GDAL `vsis3` driver's per-tile HTTP range-requests
  against MinIO are the real cost, not CPU or disk.

### gunicorn worker count (`WEB_CONCURRENCY`)
- **Plain definition:** the number of OS processes gunicorn (the Python WSGI server behind titiler)
  spawns to handle requests in parallel; requests beyond that count queue.
- **Why it shows up here:** it was the leading theory for the delay (more workers = less queueing)
  until `docker stats` showed CPU usage was far below saturation, which is inconsistent with a
  worker-queueing bottleneck.

### `docker stats` as a bottleneck-elimination tool
- **Plain definition:** live per-container CPU%/RAM view, run alongside a load test.
- **Why it shows up here:** it's the tool that actually disproved the worker theory — timing numbers
  alone were ambiguous and looked consistent with a queueing story; only concurrent resource data
  ruled it out.

## 7. Where it got stuck

**False lead 1 — "it's the HDD (D:)."**
- Symptom: user's own hypothesis — raster data lives on `D:\frims-raster-store` (HDD), viewing is
  slow, therefore move it to `C:\` (SSD).
- Investigation: timed the same 24-tile batch against a raster step never read before in the session
  (cold OS cache) vs. a step read a second time (warm cache). Cold: 661ms. Warm: 868ms — warm was
  *slower*, which is noise, not a disk-speed signal; if disk I/O were the bottleneck, warm should be
  reliably and clearly faster than cold, not statistically indistinguishable.
- Conclusion: disk speed does not explain the latency. Also, C: only has 248GB free vs. D:'s 1.83TB
  — moving data there would trade an unproven benefit for a real, near-term capacity problem.
- Status: **ruled out by measurement**, not just theorized.

**False lead 2 — "it's gunicorn worker queueing (WEB_CONCURRENCY too low)."**
- Symptom: 24 parallel tile requests taking 661-868ms looked consistent with "24 requests / 8 workers
  ≈ 3 sequential batches," especially since an old code comment already claimed 8 workers roughly
  halved latency vs. 1 worker in an earlier, unrelated test.
- Investigation: raised `WEB_CONCURRENCY` to 16 and restarted the container. First measurement after
  restart was 2716ms — much worse, but this is very likely container cold-start (JIT warmup, fresh
  worker processes, no OS page cache for that container yet), not a real 16-worker regression; it was
  not treated as signal. Two subsequent warm measurements at 16 workers came back 752ms and 662ms —
  statistically the same as the 8-worker baseline (661-868ms).
- Falsifying evidence: `docker stats --no-stream` run *during* a fresh 24-tile burst showed titiler at
  10.53% CPU and MinIO at 0.32% CPU. A worker-queueing bottleneck would show CPU pegged near 100% on
  the container while requests wait; it did not.
- Conclusion: worker count is **not** the bottleneck. Reverted `WEB_CONCURRENCY` to 8 (16 costs ~1GB
  more RAM — 2.5GB vs 1.5GB — for zero measured benefit).
- Status: **ruled out by measurement**, not just theorized.

**Unresolved — the actual cause of the ~700-900ms/batch delay is not identified.**
- This is explicitly an inference, not a confirmed finding: the leading remaining suspect is
  round-trip HTTP overhead in GDAL's `vsis3` driver, which issues multiple small byte-range requests
  per tile against MinIO, and that per-request overhead may be a roughly fixed cost independent of
  both CPU and disk speed — which would be consistent with both false leads being ruled out. This has
  **not** been verified in this session (no GDAL-layer-only measurement was taken, e.g. `gdalinfo`/
  `gdal_translate` run directly inside the titiler container to isolate GDAL time from HTTP time, or
  testing `GDAL_HTTP_VERSION=2` for request multiplexing). It is recorded as the next thing to check,
  not as a resolved root cause.

## 8. Verify

Baseline reproduction (24 tiles, z17, parallel, against the running titiler container):
```bash
# from inside services/tile-server, with the stack up (docker compose ps)
for x in $(seq 0 4); do for y in $(seq 0 4); do
  curl -s -o /dev/null "http://127.0.0.1:8090/cog/tiles/17/<x0+$x>/<y0+$y>?url=s3://flood-rasters/scenarios/depth-vl-2026-08/v1/cog/depth_0XX.tif" &
done; done
wait
```
Expected: total wall time in the 600-900ms range on this host, regardless of `WEB_CONCURRENCY` (8 or
16) and regardless of OS cache state — reproducing this range with any raster step confirms the
finding still holds before trying a new theory.

Web-side check for the `maxzoom` fix:
```bash
cd web && npx tsc --noEmit
```
Expected: no errors (already run and confirmed clean in this session). Visual check: open `/gis-map`,
zoom in past z18 on an area with the flood layer active, confirm the network tab stops issuing new
`/cog/tiles/19/...` etc. requests once past z18.

## 9. Gotchas

- Do not reuse the old `docker-compose.yml` comment's "8 workers halved latency vs 1 worker" claim as
  justification to keep tuning `WEB_CONCURRENCY` — that test was against a different bottleneck shape
  (1 worker literally serializes everything) and does not imply 8→16 helps; it doesn't, per this
  session's measurement.
- The first measurement after any `docker compose up -d` / container restart is unreliable (cold
  start) — always take at least two warm measurements before treating a number as signal. The 2716ms
  reading in false lead 2 would have looked like a severe regression if taken as the only data point.
- `maxzoom: 18` only reduces requests *above* z18 — it will not measurably change the z17 batch
  numbers used as the benchmark in this report. Don't reuse this fix as "the performance fix" in
  future reporting; it's a separate, smaller, independent improvement.
- C: (SSD) has less free space (248GB) than D: (HDD, 1.83TB) on this host as of 2026-08-17 — re-check
  before ever proposing the SSD move again, since free space changes over time and could flip the
  tradeoff.
