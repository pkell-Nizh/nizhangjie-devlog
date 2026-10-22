---
name: project-citygen-wallcity
description: "City gen M11 VIẾT LẠI 2026-07-04 — pipeline 'thành quách': trục chính 3 ô + BSP block 2 ô + hẻm 1 ô + nhà dãy packed, footprint ×2, Entrance cho NPC"
metadata: 
  node_type: memory
  type: project
  originSessionId: 31a62752-d2f8-4ffa-afd9-b402b570f4d5
---

**CITY GEN VIẾT LẠI TOÀN BỘ (2026-07-04) — pipeline "THÀNH QUÁCH"** (`World/City/Generation/`).
Bản cũ (PDS intersection + L-road + RoadTrimmer + density-gate) ĐÃ XOÁ: `RoadIntersectionPlacer`,
`RoadAnchorSampler`, `RoadTrimmer`, `SmallBuildingPlacer`, `ZonedBuildingPlacer`.
**`PoissonDiskSampler` GIỮ (Wild M12 dùng)**; `CityDensityMap` giữ chỉ cho debug heatmap + test 7/9.

**Pipeline mới (CityGenerator.Generate, 6 bước):**
1. `LandmarkPlacer.PlaceCastle` — castle 10×10 tâm map (chỉ còn castle; clan chuyển xuống block).
2. `ZoneMap` giữ nguyên (Core tròn + dải Market/Craft + Residential).
3. `RoadBuilder.BuildFrame` → `RoadPlan` (Dictionary tile→level; level 3=chính/2=phố/1=hẻm, Add giữ max):
   quảng trường quanh castle (plazaRingWidth) + **đường chính 3 ô** 1–4 trục thẳng rìa→castle
   (thứ tự N,S,E,W = cổng = MapExits) + phố vành đai 2 ô trong "tường" (`cityExtent`×min(W,H)).
4. `BlockSubdivider.MacroBlocks` — trừ strip trục + core square rồi BSP: block > `blockMaxSize`(16)
   → chẻ đường phố 2 ô, lệch tâm `splitJitter`(0.25). `AlleySplit` (gọi SAU compound): block sâu
   > `alleyDepthMax`(10) → hẻm 1 ô chẻ chiều sâu.
5. `BlockBuildingPlacer` — phân loại theo footprint: min-dim ≥ `compoundMinDim`(6) = COMPOUND
   (chiếm lô, bám mặt phố cấp cao nhất, căn giữa cạnh; clan house = compound block gần castle nhất);
   còn lại = NHÀ DÃY xếp san sát (gap 0, `vacancyChance` chừa sân) dọc cạnh block có đường, cạnh
   phố lớn pack trước. Template → "túi" quota per-zone (roll+shuffle), hết túi → filler
   residence/shop. Túi rút fail được TRẢ LẠI đầu túi (không mất quota).
6. Stamp giữ nguyên (Road tile + PlaceBuilding).

**Bất biến pathfinding (thiết kế cho pawn/NPC):** mọi đường chẻ chạy SUỐT block → mạng 4-liên-thông
theo cấu trúc (arms 2 hoặc 4 thì KHÔNG có seam; arms 1/3 có seam dead-end nhưng vẫn liên thông).
`PlacedBuilding.Entrance` = tile ĐƯỜNG trước cửa (giữa mặt phố, quét giữa→2 bên) + `HasEntrance`;
NPC pathfind tới Entrance rồi "vào". `CityLayout.RoadLevels` song song RoadTiles + `RoadLevelAt()`
(lazy dict); ctor cũ 7-arg vẫn giữ. Renderer tô 3 màu theo cấp (main sáng/alley tối).

**Footprint ×2 toàn bộ** (user yêu cầu): 41 JSON ở CẢ `Assets/Data/city_buildings/` VÀ
`StreamingAssets/Data/city_buildings/` (perl nhân đôi, giữ UTF-8). Castle 10×10, clan 6×6,
residence 4×2, housing_tier3 4×4, market_tier4 12×8, library_tier5 12×10 (compound cần block to
— best-effort, log warning nếu không vừa).

**Config mới (`CityGenConfig`, default map 128×128):** cityExtent/mainRoadWidth(3)/mainRoadArms(4)/
plazaRingWidth/streetWidth(2)/alleyWidth(1)/blockMaxSize/blockMinSize/alleyDepthMax/splitJitter/
compoundMinDim/buildingGap/vacancyChance. Field cũ (roadWidth, intersection*, anchor*, arterial*,
roadTrim*, zoneDensityFactor*, smallBuildingMinSpacing, addHorizontalExitRoad,
clanHouseMinDistFromCastle) ĐÃ XOÁ khỏi class + JSON + SO asset. Presets: large_city 400
(extent .85, plaza 4, blockMax 18), village 96 (**arms=2** chỉ trục dọc, vacancy .25, alleyDepthMax
14 → làng thưa). `large_city.json` viết lại ở cả 2 chỗ Data.

**Determinism gotchas đã né:** RoadPlan.Tiles là Dictionary → generator SORT (y,x) trước khi ra
CityLayout; BuildFillerPool sort theo id; mọi sort có tie-break index.

**Test `M11CityTest` (menu Run M11 City Tests):** defs test ×2, thêm Test 10 (BFS từ exit phủ 100%
road tiles + RoadLevels song song 1–3) + Test 11 (mọi building trừ castle có Entrance là road tile).
Đã verify COMPILE bằng Unity Roslyn (csc.dll + rsp Bee, cả Assembly-CSharp + Editor) vì Editor đang
mở không chạy được batchmode — test CHƯA CHẠY, cần user chạy menu.

**Lưu ý scene:** `CitySceneBootstrap.gridWidth/Height` default đổi 64→0 (=dùng config), nhưng
`CityScene.unity` đang serialize 200×200 + preset large_city → vào thẳng Editor = thành 200
(param 400 vẫn chạy tốt vì BSP scale-free); vào từ WorldMap thì gridWidth bị set 0 → đúng 400.
Template `_settlementTemplateJson` trong scene CHƯA wire (fileID 0) → chạy filler-only.

Docs: banner cập nhật đầu `M11_CITY_GENERATION_PIPELINE.md` (bảng map thiết kế→code) +
`M11_UNITY_IMPLEMENTATION_GUIDE.md`.

Liên quan: [[project-nizhangjie]], [[project-localmap-wildgen]], [[project-nizhangjie-gotchas]].
