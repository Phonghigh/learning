# SESSION-2026-08-17 — Flood raster "chớp tắt" khi tua timeline: quay lại kiến trúc `image` source

**Date:** 2026-08-17

**Continuation of:** `SESSION-2026-08-17-flood-raster-crossfade-fix.md` (cùng ngày, cùng câu chuyện —
đây KHÔNG phải bug mới tách biệt, đây là phần sau của việc sửa "raster ngập giật khi tua").

---

## 1. Requirement recap

Sau khi crossfade 2-slot (session trước) được triển khai, user tiếp tục báo lỗi cùng loại: "chớp
tắt, lúc hiện lúc không" khi tua nhanh. Qua 2 vòng vá thêm vẫn còn triệu chứng, và ở vòng cuối user
đưa ra tín hiệu quyết định: *"trước khi triển khai các solution này, khi dùng 20 lớp ngập nó hiển thị
rất mượt"* — so sánh trực tiếp với hành vi đã biết là đúng (kiến trúc bake-PNG + `image` source cũ,
bị bỏ đầu ngày vì lo ngại độ phân giải gốc quá lớn để nhét 1 ảnh).

## 2. How it was implemented + docs used

Diagnostic-first cho 2 vòng đầu (đọc code thật, không đoán), sau đó quyết định kiến trúc ở vòng 3
thay vì tiếp tục vá lỗi lẻ tẻ.

- Vòng 1: đọc lại `useMapContextLayers.ts`, phát hiện race condition thật trong cơ chế crossfade —
  listener `sourcedata` cũ không bị gỡ trước khi đăng ký listener mới trên cùng 1 `targetId` khi
  user đổi bước nhanh hơn tốc độ tải. Sửa bằng số thứ tự tăng dần + `map.off()` chủ động.
- Vòng 2: nhận diện `raster-fade-duration` (mặc định ~300ms của MapLibre) chồng lên cơ chế crossfade
  tự viết — 2 hiệu ứng mờ dần độc lập chạy cùng lúc, ở tốc độ x4 (250ms/bước) fade chưa xong đã sang
  bước kế. Tắt bằng `"raster-fade-duration": 0`.
- Vòng 3 (quyết định kiến trúc): thay vì tiếp tục vá thêm 1 lỗi cụ thể khác (có thể còn
  `raster-opacity-transition` mặc định cũng góp phần — CHƯA XÁC NHẬN, chỉ là suy đoán hợp lý dựa
  trên việc vòng 1+2 đều không dứt điểm), lùi lại đánh giá: `raster` tile source luôn cần nhiều
  request async phối hợp + nhiều lớp animation/transition nội bộ của MapLibre không thể tắt hết
  chắc chắn từng cái một. `image` source + `updateImage()` là thao tác THẬT SỰ atomic (1 bitmap,
  không tile, không fade) — đây là lý do kiến trúc cũ (bake-PNG) mượt.
- Insight giải quyết mâu thuẫn với lý do bỏ `image` source ban đầu (ảnh gốc 115 triệu pixel,
  13709x8382px, quá lớn để nhét 1 file): ở mức zoom hiển thị THẬT của app (12.8, đọc từ
  `app_config.map-style.zoom` trong Postgres), ảnh cần thiết chỉ ~1500x900px. Verify bằng
  `rio_tiler.io.Reader.preview(max_size=1536)`: PNG ra ~170KB, sinh trong <0.4s — hoàn toàn khả thi
  làm 1 ảnh phẳng/bước.

Options considered:
- **Tiếp tục vá `raster` tile source** (vá thêm transition/opacity còn lại) — rejected: vòng 1+2 đã
  sửa đúng 2 bug thật (có bằng chứng cụ thể) nhưng triệu chứng tương tự vẫn còn sau đó — tín hiệu
  rằng vấn đề nằm ở TẦNG kiến trúc (phối hợp nhiều async operation), không phải 1 lỗi cụ thể còn sót.
- **Giữ tile pyramid, bỏ hẳn preview ảnh** — rejected: mất khả năng atomic swap, quay lại đúng vấn đề.
- **`image` source + preview ảnh ở mức zoom hiển thị thật, giữ tile pyramid cho tương lai** (chọn) —
  atomic swap đúng như bake-PNG cũ, không mất tính năng zoom sâu sau này (tile z14-18 vẫn bake, chỉ
  không dùng ở web lúc này — YAGNI cho phần đó).

## 3. Remember card

| File | Role | Before | After |
|---|---|---|---|
| `web/src/components/map/useMapContextLayers.ts` | Effect quản lý layer raster ngập trên MapLibre | Crossfade 2-slot ping-pong (`"flood-raster-0"`/`"flood-raster-1"`) dùng `raster` tile source, đợi `sourcedata`/`isSourceLoaded` | 1 `image` source cố định id `"flood-raster"`, đổi bước gọi `updateImage({url, coordinates})` — atomic, không listener nào cần thiết |
| `web/src/lib/floodTiles.ts` | Xây URL cho web đọc raster ngập | `buildStaticFloodTileUrl()` trả URL template `{z}/{x}/{y}.png` cho tile pyramid | `buildPreviewImageUrl()` trả URL 1 file PNG phẳng/bước; thêm `FLOOD_IMAGE_COORDINATES` (4 góc AOI suy từ `FLOOD_AOI_BOUNDS_4326`) |
| `web/src/pages/GisMap.tsx` | Preload raster ngập lúc vào trang | Preload 624 tile z14 (13x12x4 bước, dùng MapLibre) | Preload 52 ảnh preview (`new Image().src`, ~9MB tổng, không qua MapLibre) |
| `services/raster-pipeline/src/raster_pipeline/tile_bake.py` | Bake tile/preview từ COG lúc ingest | `bake_step()` chỉ sinh tile z14-18, trả số tile (int) | `bake_step()` sinh CẢ tile z14-18 VÀ 1 ảnh preview (`reader.preview(max_size=1536)`) trong cùng 1 lần mở COG, trả `(số tile, số byte preview)` |
| `services/raster-pipeline/src/raster_pipeline/storage.py` | Tính key MinIO | Có `tile_prefix()`, `prefix_has_objects()` | Thêm `preview_key()` (path ảnh preview) + `object_exists()` (head_object, không so sha) |
| `services/raster-pipeline/src/raster_pipeline/pipeline.py` | Orchestrate ingest 1 file COG | Idempotent check chỉ dựa trên tile pyramid đã có chưa | Idempotent check riêng cho preview (COG cũ có tile nhưng thiếu preview → tải về bake bù, không convert lại) |

## 4. Code changes in detail

### 1. Mount effect: `raster` tile source (2 slot) → `image` source (1 id cố định) — `web/src/components/map/useMapContextLayers.ts`

**Before:**
```ts
import { buildStaticFloodTileUrl, FLOOD_AOI_BOUNDS_4326 } from "../../lib/floodTiles";
...
const visibleFloodLayerIdRef = useRef<"flood-raster-0" | "flood-raster-1">("flood-raster-0");
const pendingFloodLayerIdRef = useRef<"flood-raster-0" | "flood-raster-1" | null>(null);
...
if (showFloodRaster && floodCogKeyRef.current) {
  const tileUrl = buildStaticFloodTileUrl(floodCogKeyRef.current);
  if (!tileUrl) {
    console.warn("flood raster: chưa cấu hình VITE_MINIO_PUBLIC_URL hoặc cog_key sai định dạng, bỏ qua");
  } else {
    const initialId = visibleFloodLayerIdRef.current;
    map.addSource(initialId, { type: "raster", tiles: [tileUrl], tileSize: 256, maxzoom: 18, bounds: FLOOD_AOI_BOUNDS_4326 });
    map.addLayer({
      id: initialId, type: "raster", source: initialId,
      paint: { "raster-opacity": floodOpacity },
    }, "rivers-line");
  }
}
```

**After:**
```ts
import { buildPreviewImageUrl, FLOOD_IMAGE_COORDINATES } from "../../lib/floodTiles";
...
// (không còn 2 ref slot ping-pong)
...
if (showFloodRaster && floodCogKeyRef.current) {
  const url = buildPreviewImageUrl(floodCogKeyRef.current);
  if (!url) {
    console.warn("flood raster: chưa cấu hình VITE_MINIO_PUBLIC_URL hoặc cog_key sai định dạng, bỏ qua");
  } else {
    map.addSource("flood-raster", { type: "image", url, coordinates: FLOOD_IMAGE_COORDINATES });
    map.addLayer({
      id: "flood-raster", type: "raster", source: "flood-raster",
      paint: { "raster-opacity": floodOpacity },
    }, "rivers-line");
  }
}
```

**What changed:** `type: "raster"` (tile URL template + `tiles: [tileUrl]`) → `type: "image"` (1
`url` + 4 `coordinates`); 2 slot ref bị xóa hoàn toàn (dead code sau khi kiến trúc đổi); id trở lại
1 chuỗi cố định `"flood-raster"`.

**Why:** `raster` source cần MapLibre chia viewport thành tile x/y/z và gửi nhiều request song song
— nguồn gốc của mọi vấn đề phối hợp async ở 2 vòng vá trước. `image` source chỉ có 1 bitmap.

**How it behaves now:** layer raster ngập luôn có đúng 1 id, không còn khái niệm "slot đang tải" vs
"slot đang hiển thị" — không có gì để đồng bộ sai nữa.

### 2. Step-change effect: gỡ toàn bộ cơ chế crossfade + `sourcedata` listener, thay bằng `updateImage()` — `web/src/components/map/useMapContextLayers.ts`

**Before (rút gọn, phần lõi):**
```ts
const targetId = pendingFloodLayerIdRef.current ?? (visibleId === "flood-raster-0" ? "flood-raster-1" : "flood-raster-0");
if (map.getLayer(targetId)) map.removeLayer(targetId);
if (map.getSource(targetId)) map.removeSource(targetId);

map.addSource(targetId, { type: "raster", tiles: [tileUrl], tileSize: 256, maxzoom: 18, bounds: FLOOD_AOI_BOUNDS_4326 });
map.addLayer({
  id: targetId, type: "raster", source: targetId,
  paint: { "raster-opacity": floodOpacity },
}, map.getLayer("rivers-line") ? "rivers-line" : undefined);
pendingFloodLayerIdRef.current = targetId;

const onSourceData = (e: maplibregl.MapSourceDataEvent) => {
  if (e.sourceId !== targetId || !e.isSourceLoaded) return;
  map.off("sourcedata", onSourceData);
  if (pendingFloodLayerIdRef.current !== targetId) return;
  if (visibleFloodLayerIdRef.current !== targetId) {
    const old = visibleFloodLayerIdRef.current;
    if (map.getLayer(old)) map.removeLayer(old);
    if (map.getSource(old)) map.removeSource(old);
  }
  visibleFloodLayerIdRef.current = targetId;
  pendingFloodLayerIdRef.current = null;
};
map.on("sourcedata", onSourceData);
```

**After:**
```ts
const url = buildPreviewImageUrl(floodCogKey);
if (!url) return;
const src = map.getSource("flood-raster") as maplibregl.ImageSource | undefined;
if (src) {
  src.updateImage({ url, coordinates: FLOOD_IMAGE_COORDINATES });
} else {
  // Chưa từng có (vd `onLoad` bỏ qua vì thiếu cấu hình lúc mount,
  // giờ đã có) -thêm mới thay vì update.
  map.addSource("flood-raster", { type: "image", url, coordinates: FLOOD_IMAGE_COORDINATES });
  map.addLayer({
    id: "flood-raster", type: "raster", source: "flood-raster",
    paint: { "raster-opacity": floodOpacity },
  }, map.getLayer("rivers-line") ? "rivers-line" : undefined);
}
```

**What changed:** toàn bộ logic remove/add slot + đăng ký/gỡ listener `sourcedata` + guard số thứ
tự bị xóa; thay bằng 1 lệnh `updateImage()` (hoặc `addSource`/`addLayer` lần đầu nếu chưa tồn tại).

**Why:** đây là fix trực tiếp cho triệu chứng ở vòng 3 — `updateImage()` là API atomic thật sự của
MapLibre cho `image` source, không có khoảng trống giữa "ảnh cũ biến mất" và "ảnh mới xuất hiện",
và không có fade/transition nội bộ nào để canh thời điểm.

**How it behaves now:** đổi bước timeline chỉ gọi 1 lệnh, không còn race condition có thể xảy ra
(không có 2 request nào để lệch nhau), không còn phụ thuộc vào việc tắt đúng hết mọi transition
property của MapLibre.

### 3. `buildStaticFloodTileUrl` (tile template) → `buildPreviewImageUrl` (1 file phẳng) — `web/src/lib/floodTiles.ts`

**Before:**
```ts
export function buildStaticFloodTileUrl(cogKey: string): string | null {
  const base = minioPublicBase();
  if (!base) return null;
  const match = cogKey.match(/^(.*)\/cog\/depth_(\d+)\.tif$/);
  if (!match) return null;
  const [, prefix, stepIndex] = match;
  return `${base}/${BUCKET}/${prefix}/tiles/${stepIndex}/{z}/{x}/{y}.png`;
}
```

**After:**
```ts
export const FLOOD_IMAGE_COORDINATES: [[number, number], [number, number], [number, number], [number, number]] = [
  [FLOOD_AOI_BOUNDS_4326[0], FLOOD_AOI_BOUNDS_4326[3]],
  [FLOOD_AOI_BOUNDS_4326[2], FLOOD_AOI_BOUNDS_4326[3]],
  [FLOOD_AOI_BOUNDS_4326[2], FLOOD_AOI_BOUNDS_4326[1]],
  [FLOOD_AOI_BOUNDS_4326[0], FLOOD_AOI_BOUNDS_4326[1]],
];

export function buildPreviewImageUrl(cogKey: string): string | null {
  const base = minioPublicBase();
  if (!base) return null;
  const match = cogKey.match(/^(.*)\/cog\/depth_(\d+)\.tif$/);
  if (!match) return null;
  const [, prefix, stepIndex] = match;
  return `${base}/${BUCKET}/${prefix}/preview/depth_${stepIndex}.png`;
}
```

**What changed:** hàm đổi tên và trả về 1 URL file cụ thể thay vì URL template có `{z}/{x}/{y}`;
thêm hằng số `FLOOD_IMAGE_COORDINATES` (4 góc AOI, thứ tự MapLibre `image` source yêu cầu:
trên-trái, trên-phải, dưới-phải, dưới-trái) suy trực tiếp từ `FLOOD_AOI_BOUNDS_4326` đã có sẵn.

**Why:** `image` source cần 1 URL ảnh + tọa độ 4 góc, không phải URL template theo tile index.

**How it behaves now:** mỗi bước thời gian ứng với đúng 1 request ảnh (thay vì nhiều request tile
tùy viewport).

### 4. Preview backfill trong pipeline bake — `services/raster-pipeline/src/raster_pipeline/tile_bake.py`

**Before:**
```python
def bake_step(client, local_cog_path: str, tile_prefix: str) -> int:
    """Sinh + upload toàn bộ tile z14-18 cho MỘT bước, từ file COG cục bộ. ..."""
    count = 0
    with Reader(local_cog_path) as reader:
        for zoom in range(14, 19):
            for tile in _tiles_for_bounds(reader, zoom):
                img = reader.tile(tile.x, tile.y, tile.z)
                png_bytes = img.render(img_format="PNG", colormap=TILE_COLORMAP)
                key = f"{tile_prefix}/{tile.z}/{tile.x}/{tile.y}.png"
                storage.upload_bytes(client, png_bytes, key, "image/png")
                count += 1
    return count
```

**After:**
```python
PREVIEW_MAX_SIZE = 1536

def bake_step(client, local_cog_path: str, tile_prefix: str, preview_key: str) -> tuple:
    """Sinh + upload toàn bộ tile z14-18 VÀ 1 ảnh preview toàn AOI, từ CÙNG
    một lần mở file COG cục bộ. Trả `(số tile, số byte ảnh preview)`. ..."""
    count = 0
    with Reader(local_cog_path) as reader:
        for zoom in range(14, 19):
            for tile in _tiles_for_bounds(reader, zoom):
                img = reader.tile(tile.x, tile.y, tile.z)
                png_bytes = img.render(img_format="PNG", colormap=TILE_COLORMAP)
                key = f"{tile_prefix}/{tile.z}/{tile.x}/{tile.y}.png"
                storage.upload_bytes(client, png_bytes, key, "image/png")
                count += 1

        preview_img = reader.preview(max_size=PREVIEW_MAX_SIZE)
        preview_bytes = preview_img.render(img_format="PNG", colormap=TILE_COLORMAP)
        storage.upload_bytes(client, preview_bytes, preview_key, "image/png")

    return count, len(preview_bytes)
```

**What changed:** thêm tham số `preview_key`, thêm 1 khối `reader.preview()` sau vòng lặp tile
(vẫn trong cùng `with Reader(...)` — không mở lại COG lần 2), đổi return type từ `int` sang `tuple`.

**Why:** giữ tile pyramid cho tính năng zoom sâu tương lai (YAGNI cho việc dùng ngay, nhưng không
xóa hạ tầng đã có) trong khi thêm sản phẩm mới (ảnh preview) mà web thực sự cần bây giờ, không tốn
thêm 1 lần mở/decode COG riêng.

**How it behaves now:** mỗi lần ingest 1 file COG mới, pipeline sinh cả tile lẫn preview trong 1
lượt; các COG cũ (nạp trước khi có tính năng preview) được bake bù preview riêng khi phát hiện thiếu
(xem thay đổi #5).

### 5. Idempotent check tách riêng cho tile vs preview — `services/raster-pipeline/src/raster_pipeline/pipeline.py`

**Before:**
```python
if storage.object_matches(client, key, sha):
    if storage.prefix_has_objects(client, tile_prefix):
        return _step_record(f, cfg, key, sha, None), f"{label} -> đã có (COG + tile), bỏ qua"
    try:
        storage.download_cog(client, key, tmp_cog)
        n_tiles = tile_bake.bake_step(client, tmp_cog, tile_prefix)
    finally:
        if os.path.exists(tmp_cog):
            os.remove(tmp_cog)
    return _step_record(f, cfg, key, sha, None), f"{label} -> đã có COG, {n_tiles} tile (bake bù)"
```

**After:**
```python
if storage.object_matches(client, key, sha):
    if storage.prefix_has_objects(client, tile_prefix) and storage.object_exists(client, preview_key):
        return _step_record(f, cfg, key, sha, None), f"{label} -> đã có (COG + tile + preview), bỏ qua"
    try:
        storage.download_cog(client, key, tmp_cog)
        n_tiles, preview_bytes = tile_bake.bake_step(client, tmp_cog, tile_prefix, preview_key)
    finally:
        if os.path.exists(tmp_cog):
            os.remove(tmp_cog)
    return (_step_record(f, cfg, key, sha, None),
            f"{label} -> đã có COG, {n_tiles} tile + preview {preview_bytes // 1024}KB (bake bù)")
```

**What changed:** điều kiện "đã đủ, bỏ qua" giờ kiểm tra CẢ tile lẫn preview (`and
storage.object_exists(client, preview_key)`), không chỉ tile.

**Why:** COG cũ (nạp trước session này) đã có tile nhưng chưa có preview — nếu chỉ check tile, các
bước này sẽ bị coi là "đã xong" và không bao giờ có preview, khiến web load ảnh 404.

**How it behaves now:** với 52 bước hiện có, pipeline tải COG về tạm (không convert lại, `sha` đã
khớp) chỉ để bake bù preview còn thiếu — đúng nhánh đang chạy nền lúc viết report này (`ingest-preview`,
xem §8).

### 6. Preload: 624 tile z14 → 52 ảnh preview — `web/src/pages/GisMap.tsx`

**Before:** (không có đoạn preload ảnh preview; trước đó preload dùng MapLibre để mồi tile z14 —
đã bị xóa cùng lúc với cơ chế `raster` tile source)

**After:**
```tsx
useEffect(() => {
  for (const s of floodSteps) {
    const url = buildPreviewImageUrl(s.cogKey);
    if (!url) break; // chưa cấu hình VITE_MINIO_PUBLIC_URL -bỏ qua toàn bộ, không log lỗi lặp 52 lần
    new Image().src = url;
  }
}, [floodSteps]);
```

**What changed:** vòng lặp preload đổi từ mồi 624 tile (13 x 12 vị trí x 4 bước, qua MapLibre) sang
mồi 52 ảnh preview (1 ảnh/bước, qua `Image` object thô, không qua MapLibre).

**Why:** đơn giản hơn nhiều (52 request thay vì 624), tổng dung lượng ~9MB, không cần MapLibre biết
gì về các ảnh này cho tới khi thật sự cần hiện — chỉ mồi cache HTTP + decode của trình duyệt.

**How it behaves now:** ngay khi vào trang `/gis-map`, toàn bộ ảnh preview được tải ngầm; từ đó tua
timeline chỉ đọc cache trình duyệt, không phụ thuộc mạng lúc đổi bước.

## 5. How to find this again

- `grep -rn "buildPreviewImageUrl\|FLOOD_IMAGE_COORDINATES" web/src`
- `grep -n "updateImage\|ImageSource" web/src/components/map/useMapContextLayers.ts`
- `grep -n "PREVIEW_MAX_SIZE\|preview_key\|object_exists" services/raster-pipeline/src/raster_pipeline`
- Route: `/gis-map`. Container backfill đang chạy: `docker ps` → `ingest-preview`.
- Session trước (bối cảnh, KHÔNG đọc nhầm là fix cuối cùng):
  `SESSION-2026-08-17-flood-raster-crossfade-fix.md` — đã bị chính session này thay thế hoàn toàn
  về mặt kiến trúc (crossfade 2-slot không còn tồn tại trong code).

## 6. Concepts introduced

### `image` source's `updateImage()` là API atomic thật sự
- **Plain definition:** MapLibre `image` source giữ đúng 1 bitmap; `updateImage({url, coordinates})`
  thay bitmap đó trong 1 lệnh, không có bước trung gian nào hiển thị "trống".
- **Why it shows up here:** đây là lý do gốc kiến trúc bake-PNG cũ mượt và là điểm khác biệt cốt lõi
  so với `raster` tile source (luôn cần nhiều tile request async + fade/transition nội bộ).

### `raster-fade-duration` là hành vi mặc định, không phải bug tự viết
- **Plain definition:** property paint mặc định (~300ms) của MapLibre khiến MỌI tile mới tải xong
  tự động mờ dần từ trong suốt lên full opacity, độc lập với bất kỳ animation nào ứng dụng tự viết.
- **Why it shows up here:** vòng 2 chồng 2 hiệu ứng mờ dần (crossfade tự viết + fade mặc định) cùng
  lúc, gây "chớp tắt" ở tốc độ tua nhanh khi fade mặc định chưa kịp hoàn tất.

### `Reader.preview()` (rio-tiler) sinh ảnh ở mức zoom hiển thị thật, không phải độ phân giải gốc
- **Plain definition:** hàm của rio-tiler đọc COG và trả về 1 ảnh đã resample xuống kích thước tối
  đa cho trước (`max_size`), khác với đọc tile gốc từng phần.
- **Why it shows up here:** insight giải quyết mâu thuẫn giữa "ảnh gốc 115 triệu pixel quá lớn" (lý
  do bỏ `image` source ban đầu) và "cần 1 ảnh phẳng để atomic swap" — ảnh ở mức zoom app thực tế
  hiển thị (12.8) chỉ ~1500x900px, nhỏ hơn nhiều so với ảnh gốc.

## 7. Where it got stuck

**Vòng 1 — race condition trong listener `sourcedata` của cơ chế crossfade (sửa đúng 1 bug thật,
KHÔNG giải quyết triệt để).**
- Symptom: user báo "chớp tắt, lúc hiện lúc không" khi tua nhanh (x4).
- False lead ban đầu (user's own guess): do gỡ layer mới/xóa layer cũ không kịp — hướng đúng đại
  khái nhưng chưa chỉ đúng cơ chế cụ thể.
- Investigation: đọc lại code crossfade trong `useMapContextLayers.ts`. Khi đổi bước NHANH HƠN tốc
  độ tải (tái sử dụng lại đúng 1 slot nhiều lần liên tiếp), listener `sourcedata` CŨ không được gỡ
  trước khi đăng ký listener MỚI trên CÙNG 1 `targetId` — guard `e.sourceId !== targetId` không
  phân biệt được "lần tải cũ của chính slot này" với "lần tải mới nhất" vì cả hai chung 1 id. Nhiều
  listener chồng lên nhau, listener cũ (lỗi thời) vẫn có thể bắn `isSourceLoaded` và bật/tắt opacity
  sai lúc — đây là bug thật, xác nhận bằng đọc code, không phải suy đoán.
- Fix: thêm số thứ tự (`floodRequestSeqRef`, tăng dần mỗi lần bắt đầu tải) + luôn `map.off()`
  listener đang treo trước khi đăng ký cái mới (`pendingFloodListenerRef`).
- Verify: `npx tsc --noEmit` + `npm run build` sạch. KHÔNG verify được bằng mắt trên trình duyệt
  thật (không có access trong môi trường này).
- **Kết quả:** sửa đúng, nhưng user vẫn báo triệu chứng tương tự sau đó — tín hiệu vấn đề chưa hết.

**Vòng 2 — `raster-fade-duration` mặc định chồng lên crossfade tự viết (sửa đúng 1 hành vi thật của
MapLibre, VẪN KHÔNG giải quyết triệt để).**
- Symptom: user mô tả rõ hơn — "lớp raster ngập cũ từ từ mờ đi rồi nó hiện lớp raster mới" (fade
  chậm, không phải chớp tức thời).
- Investigation: nhận diện đây là hành vi MẶC ĐỊNH của MapLibre (`raster-fade-duration` ~300ms),
  không phải bug tự viết — mọi tile mới tải xong tự động mờ dần lên full opacity, chồng lên cơ chế
  crossfade 2-lớp tự viết. Ở x4 (250ms/bước), fade 300ms chưa xong đã sang bước kế → luôn ở trạng
  thái mờ dở dang.
- Fix: thêm `"raster-fade-duration": 0` vào paint của cả 3 layer raster ngập (layer chính lúc mount,
  layer trong crossfade, layer prefetch).
- Verify: build sạch, KHÔNG verify mắt được.
- **Kết quả:** sửa đúng 1 hành vi thật, nhưng user vẫn báo chớp tắt sau đó.

**Vòng 3 — quyết định kiến trúc (KHÔNG phải vá lỗi cụ thể), dựa trên tín hiệu so sánh trực tiếp của
user.**
- Symptom: user vẫn báo chớp tắt sau khi đã sửa vòng 1+2, và chỉ ra thẳng: "trước khi triển khai các
  solution này, khi dùng 20 lớp ngập nó hiển thị rất mượt" — so sánh trực tiếp với kiến trúc CŨ
  (bake-PNG + `image` source).
- Investigation/reasoning: thay vì tiếp tục vá thêm 1 lỗi cụ thể khác (có thể còn
  `raster-opacity-transition` mặc định cũng góp phần — **suy đoán, CHƯA XÁC NHẬN bằng đo đạc trực
  tiếp**, chỉ dựa trên việc vòng 1+2 đều không dứt điểm hoàn toàn dù mỗi lần đều sửa đúng 1 bug
  thật), quyết định lùi lại đánh giá kiến trúc tổng thể: `raster` tile source luôn cần nhiều request
  async phối hợp với nhau + có nhiều lớp animation/transition nội bộ của MapLibre (fade-duration,
  opacity-transition...) không thể tắt hết từng cái một một cách chắc chắn. `image` source +
  `updateImage()` là thao tác THẬT SỰ atomic — đây chính là lý do kiến trúc cũ mượt.
- Insight giải quyết mâu thuẫn: ảnh gốc 115 triệu pixel quá lớn cho 1 file, nhưng ở mức zoom hiển
  thị THẬT của app (12.8), ảnh cần thiết chỉ ~1500x900px — verify bằng `rio_tiler...Reader.preview()`
  thật, không phải suy đoán (PNG 173KB, 0.34s, đo trực tiếp).
- Fix: viết lại kiến trúc — xem §4 (toàn bộ 6 thay đổi). Giữ tile pyramid z14-18 cho tương lai, thêm
  ảnh preview làm nguồn hiển thị chính.
- **Bài học meta:** 2 lần đầu đều là sửa đúng bug thật (không phải sai — có bằng chứng cụ thể: race
  condition thật đọc thấy trong code, `raster-fade-duration` là hành vi mặc định thật của MapLibre)
  — nhưng việc "vẫn còn triệu chứng tương tự sau khi sửa bug thật" là tín hiệu quan trọng để LÙI LẠI
  đánh giá kiến trúc thay vì tiếp tục vá từng lỗi nhỏ lẻ trong 1 cơ chế vốn dĩ phức tạp hơn cần thiết
  (raster tile source đòi hỏi phối hợp nhiều async operation cho 1 việc mà về bản chất chỉ cần thay
  1 ảnh). User đưa ra tín hiệu quyết định bằng cách so sánh trực tiếp với hành vi đã biết là ĐÚNG
  (kiến trúc cũ mượt) — đây là cách hiệu quả để phát hiện "đang sửa sai tầng vấn đề".

## 8. Verify

```bash
cd web
npx tsc --noEmit -p .
npm run build
```
Expected: cả hai exit 0. Đã chạy lại trong khi viết report này — `tsc` sạch, không lỗi.

Pipeline bake preview backfill (Docker, chạy thật, không phải mock):
```bash
docker compose run ingest   # nhánh "COG+tile đã có, thiếu preview -tải về bake bù"
docker stats --no-stream
```
Xác nhận qua `docker stats` lúc viết report: container `ingest-preview` đang chạy CPU ~197%,
Memory ~5GB, NET I/O tăng liên tục — đang xử lý thật, không treo.

**PHIÊN NÀY VẪN ĐANG CHẠY NỀN LÚC VIẾT REPORT, CHƯA CÓ KẾT QUẢ CUỐI CÙNG.** Đang chờ pipeline bake
xong preview cho 52 bước hiện có. **Chưa verify bằng mắt trên trình duyệt thật** — cần mở `/gis-map`
sau khi backfill hoàn tất, tua timeline (bao gồm tua nhanh x4) và xác nhận không còn chớp tắt, ảnh
đổi mượt như kiến trúc bake-PNG cũ mà user đã dùng làm chuẩn so sánh.

## 9. Gotchas

- Nếu code nào đó còn hardcode 2 id cũ (`"flood-raster-0"`/`"flood-raster-1"`) hoặc còn gọi
  `buildStaticFloodTileUrl` (đã xóa khỏi `floodTiles.ts`), sẽ lỗi biên dịch ngay (không phải lỗi
  runtime im lặng) — `tsc --noEmit` đã xác nhận sạch nên không còn tham chiếu nào sót lại tính đến
  thời điểm viết report.
- Tile pyramid z14-18 vẫn được bake ở Python nhưng KHÔNG còn được web dùng — nếu sau này dọn dẹp
  pipeline "cho gọn", đừng xóa nhầm phần bake tile nghĩ rằng nó dư thừa; đây là hạ tầng giữ lại có
  chủ đích cho tính năng zoom sâu tương lai (đọc doc-block trong `tile_bake.py` trước khi xóa).
- `object_exists()` (mới thêm ở `storage.py`) chỉ kiểm tra object có tồn tại, KHÔNG so sánh checksum
  như `object_matches()` — nếu ảnh preview bị hỏng/cũ nhưng vẫn tồn tại ở đúng key, pipeline sẽ coi
  là "đã xong" và không bake lại. Muốn ép bake lại preview phải xóa object đó thủ công trên MinIO.
- `FLOOD_IMAGE_COORDINATES` suy trực tiếp từ `FLOOD_AOI_BOUNDS_4326` — nếu sau này AOI đổi (nhiều
  kịch bản khác bbox), phải cập nhật cả hai cùng lúc; hiện đang hardcode 1 kịch bản (YAGNI, đã ghi
  chú sẵn trong `floodTiles.ts`).
- Việc dang dở khác trong ngày (Pha 6 hoãn, git filter-repo hoãn, verify QGIS thật) không liên quan
  đến session này.
