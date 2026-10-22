# M10 v2 — Redesign theo prototype FastNoiseLite

Bản nâng cấp pipeline sinh world dựa trên prototype Raylib của bạn. Map cũ "tệ hại" vì dùng `Mathf.PerlinNoise` (lặp pattern, đối xứng trục, không seed được). Bản này thay bằng **FastNoiseLite (OpenSimplex2)** + các bước hậu xử lý từ prototype, **giữ nguyên** phần đặc thù dự án mà prototype không có.

---

## 0. ⚠️ BẮT BUỘC TRƯỚC TIÊN — thêm FastNoiseLite.cs

Code mới phụ thuộc class `FastNoiseLite`. **Chưa có file này thì cả project không compile.**

1. Lấy file **`FastNoiseLite.cs`** (chính cái bạn đang dùng trong prototype Raylib — bản C# single-file, public domain).
2. Bỏ vào **`Assets/Scripts/ThirdParty/FastNoiseLite.cs`** (hoặc bất kỳ đâu trong `Assets/`).
3. Đảm bảo nó để ở **global namespace** (mặc định không có `namespace`) — code dự án gọi `FastNoiseLite` không qualifier, đúng như prototype của bạn.
4. KHÔNG cần file Raylib — chỉ riêng `FastNoiseLite.cs`.

Sau khi thêm → Unity compile lại → chạy `Nizhangjie/WorldGen/Preview Texture`.

---

## 1. Khảo sát prototype: giữ gì, bỏ gì

| Thành phần prototype | Quyết định |
|---|---|
| FastNoiseLite OpenSimplex2 (FBm + Ridged) | **Lấy** — thay Mathf.PerlinNoise. Đây là fix lớn nhất. |
| Domain warp (FNL DomainWarpProgressive) | **Lấy** — bờ biển/dãy núi hữu cơ. |
| Nhiệt = gradient vĩ độ + noise − lapse | **Lấy** (mình đã có, nay tinh chỉnh theo prototype). |
| Rain shadow (gió Tây→Đông, núi chắn mưa) | **Lấy** — biome thực tế, sa mạc sau núi. |
| Whittaker classify (snow/tundra/desert/forest/rainforest) | **Lấy** — bổ sung biome `forest`, `tundra` còn thiếu. |
| Cellular automata smoothing | **Lấy** — xoá nhiễu viền biome. |
| Lọc vùng nhỏ (flood fill thay biome) | **Lấy** — bản đồ gọn. |
| Đặt vùng "Special" ngẫu nhiên | **Bỏ** — dự án đã có Tiên Cảnh/Ma Vực gắn linh mạch (tốt hơn). |
| Render Raylib + Camera2D | **Bỏ** — dự án dùng Unity Tilemap (`WorldMapRenderer` đã có). |

**Giữ riêng của dự án (prototype không có):** sông downhill (`HydrologyPass`), linh mạch + long nhãn (`SpiritVeinPass`), Tiên Cảnh/Ma Vực gắn mạch (`SpecialZonePass`), `LingQiDensity`, hệ 2 tầng World→Local.

---

## 2. Pipeline mới (9 pass, đã code)

```
1 Elevation       FastNoiseLite FBm×0.65 + Ridged×0.35, domain warp
2 Climate         nhiệt (vĩ độ+noise−lapse) + ẩm + RAIN SHADOW
3 Mountain        đánh dấu IsMountain (ridge đã nằm trong elevation) + lọc cụm
4 Hydrology       sông downhill (giữ nguyên — prototype không có)
5 Biome           Whittaker: water/snow*/tundra/desert/plains/forest/rainforest/mountain*
6 Smooth          cellular automata majority (giữ nước)        ← MỚI từ prototype
7 RegionCleanup   nuốt vùng biome < minRegionSize              ← MỚI từ prototype
8 SpiritVein      linh mạch Bezier + long nhãn (giữ nguyên)
9 SpecialZone     Tiên Cảnh/Ma Vực gắn linh khí (giữ nguyên)
```

File thay đổi: `ElevationPass`, `ClimatePass`, `MountainPass`, `BiomePass` (viết lại) + `SmoothPass`, `BiomeRegionCleanupPass` (mới) + `WorldGenerator` (thứ tự) + `WorldGenConfig` (tham số FNL) + `BiomeDefinition` (thêm forest/tundra). `FastNoise.cs` (Hash) giữ lại cho `SpecialZone`/`SpiritVein`/localSeed.

---

## 3. Thay đổi tham số quan trọng (worldgen_config.json)

Tần số giờ là **per-pixel** (FNL), không nhân theo map nữa:

| Tham số | Cũ (Perlin) | Mới (FNL) | Ý nghĩa |
|---|---|---|---|
| `elevation.frequency` | 3.5 | **0.012** | nhỏ hơn = lục địa to hơn |
| `elevation.warpAmplitude` | 0.25 | **35** | độ méo domain warp (px) |
| `elevation.ridgeBlend` | — | **0.35** | tỉ lệ trộn ridged (núi sắc) |
| `seaLevel` | 0.30 | **0.32** | mực biển |
| `mountainLevel` | 0.60 | **0.72** | ngưỡng núi |
| `temperature.frequency` | — | **0.010** | — |
| `moisture.frequency` | — | **0.014** | — |
| `rainShadow.*` | — | **mới** | núi chắn mưa |
| `smooth.iterations` | — | **2** | số lần làm mượt |
| `regionCleanup.minRegionSize` | — | **40** | vùng nhỏ hơn bị nuốt |

Toàn bộ vẫn deterministic theo `seed`. Đổi JSON → chạy lại Preview, không cần restart.

---

## 4. Tune để map đẹp (nếu vẫn chưa ưng)

| Map bị… | Chỉnh |
|---|---|
| Lục địa quá nhỏ/vụn | giảm `elevation.frequency` (vd 0.008) |
| Bờ biển quá tròn | tăng `elevation.warpAmplitude` (vd 50) |
| Núi quá ít | giảm `mountainLevel` (vd 0.68) |
| Núi rời rạc | tăng `mountain.minRangeSize` |
| Biome lốm đốm | tăng `smooth.iterations` (3) hoặc `regionCleanup.minRegionSize` (60) |
| Quá nhiều sa mạc sau núi | giảm `rainShadow.mountainFactor` về 0.7 (chắn ít hơn) |
| Sa mạc/tuyết quá nhiều | chỉnh `climateThresholds` (vd `tempDesertMoist` xuống 0.2) |

---

## 5. Lưu ý

- **Test M10**: `Nizhangjie/Run M10 Tests` vẫn chạy. Trên map test nhỏ (96×96) với ngưỡng mới cao hơn, sông có thể ít — nếu test "tồn tại sông" fail, hạ `river.minSourceElevation` (vd 0.6) hoặc tăng map test. Không phải lỗi compile.
- **Preview & Scene**: `WorldGenPreview`, `WorldMapRenderer`, HUD **không đổi** — chúng đọc `BiomeId`, tự hiểu 2 biome mới (forest/tundra) qua màu tile trội.
- **Biome mới**: đã thêm `biome_forest.json`, `biome_tundra.json` vào `Resources/Biomes/` và vào `BiomeDefinition.DefaultSet()`.
- **Determinism**: FastNoiseLite seed bằng int → cùng platform cho kết quả y hệt (tốt hơn Mathf).

---

## 6. Quan hệ tài liệu

- **`M10_REDESIGN_FASTNOISE.md`** (file này) — bản nâng cấp v2, đọc TRƯỚC.
- `M10_UNITY_EDITOR_GUIDE.md` — chạy preview/test (vẫn đúng, chỉ khác file noise).
- `M10_WORLDMAP_SCENE_GUIDE.md` — dựng scene tương tác (không đổi).
- `M10_WORLDGEN_PIPELINE.md` — thiết kế tổng quan.

---

*Tóm tắt: thêm `FastNoiseLite.cs` → compile → Preview. Map sẽ đẹp hơn hẳn nhờ OpenSimplex2 + domain warp + rain shadow + smoothing.*
