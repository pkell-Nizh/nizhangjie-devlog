---
name: project-chunk-commit
description: "Ô world 240 + nạp/gỡ theo chunk + cắt khựng qua ranh (2026-10-15/16, commit 9bb8b92 + 578d02f) — cơ chế, số đo, phần còn lại, bẫy đo"
metadata:
  node_type: memory
  type: project
  originSessionId: 5ba4247b-b963-444b-9a01-bc1d05820df6
  modified: 2026-10-15T22:49:56.218Z
---

User chốt "cách lai" (tạo theo ô world ở luồng nền, đưa vào/ra game theo chunk) và **cạnh ô world = 240** (15 chunk 16 × 20 section tìm đường 12; section = số lớn nhất ≤ 12 chia hết cạnh ô — 200 cho 10). Save cũ theo khoá ô world sẽ lệch — user chấp nhận.

**Cơ chế (đã commit):**
- `VoxelWorld._commitQueue` (3 ms/khung, gần avatar trước, chunk trong `NearCommitChunks`=2 làm ngay); `DesiredType` tra số; chụp ô 1/khung.
- `VoxelGameView`: `_spawnQueue` (bản dựng đầu theo chunk), `_destroyQueue` (12 chunk/khung); dựng lại theo ánh sáng / hang theo **ms** (2,5) chứ không theo số chunk.
- `SectorWindow.Tick` gỡ 1 ô cũ/lượt + `FreeSlotFor`.
- `PathRegionGrid.DeferRectSections` = **2** (từ 64): vùng đổi độ cao ≥ 2 section vào sổ dựng dần — trước đó mỗi chunk commit làm bẩn section và câu hỏi đường kế tiếp dựng lại hết (~80–170 ms/câu của quái). Sửa 1 khối vẫn dựng ngay.
- `BeastBehaviour.OnTick` ngân sách 3 ms/tick. 2D tilemap hoãn vẽ/xoá khi `WorldViewOverride.OwnsCamera`.
- Marker: `VW.* SW.* VGV.* LMD.* PG/PF/PH.* PRG.* RP.* Beast.* Clock.<Lớp>.<Hàm>` (GameClock mẫu riêng từng hàm nghe).

**Số đo qua ranh (3D, 240, Editor remote):** ApplyBuilt 85→1,2; Destroy 101→2; VW.Apply 192→≤8; gỡ ô 143 (5/khung)→10–14/ô; Beast OnTick 166→29; khung tệ nhất ~190→~100 (lẻ), p95 ~30.
**dff8b7c:** `RegionReachability` BFS hai chiều + nhãn vùng đã loang hết (RP.Reach 27→8); `PathGrid.RecomputeBlock` duyệt dải chỉ số + `GridSystem.AtIndex` đọc ref (Merge 12–14→4). Qua ranh sau đợt này: p95 ~28, p99 ~38, tệ nhất ~96.
**08c3526 (ô có thành):** CityZoneOverlay hoãn vẽ khi 3D giữ camera (193 ms); WildSceneInteraction.RebuildScanner cache def (127 ms đọc JSON); CityNpc tìm đường trong nhà ngân sách 4 ms/SlowTick (207); BlockTicks cache section theo ô; BeastSpawner vẽ thú qua hàng đợi 2 ms; WildTilemapRenderer không xoá đồng bộ ô nạp lại khi 3D; RP trả null khi section xuất phát đang chờ dựng. Nạp lại ô thành sau đợt này: không City.*/Layout.* nào ≥3 ms; p95 ~11, p99 ~19; còn GC ~60, một câu A* của quái 12–22, VW.Ticks 5–6.
**Còn lại (trước 08c3526):** `SW.RemoveSector` ~10 (Lists ~4,5 RemoveAll cả cửa sổ; PRG gỡ region ~4); `VW.Want` ~10–12; `VW.Ticks` 2–14/khung; ô có thành: `OnCitySectorUnloaded` ~131, Merge ô thành ~110, `CityNpc.SlowTick` tới 220 lúc thành mới nạp. Test `ContinuousWildMapTest.Mountain_EveryRockTileHasMineableWall` đỏ có sẵn từ f6e2439.

**Bẫy đo:** Editor mất focus ⇒ gần như không chạy khung — `Application.runInBackground = true` qua execute_code SAU khi domain reload xong (gọi ngay lúc vào Play thì lỗi). `ProfilerDriver.profileEditor = false`, ClearAllFrames + bật + ForceCell trong CÙNG lệnh, đọc sau ~5 s (bộ đệm 2000 khung trôi nhanh ở 3D). Lần đầu một ô có thành: Mono.JIT vài ms.

Liên quan: [[project-tick-spread]], [[project-mc-move-speeds]].
