---
name: project-load-order-view-circle
description: "Vào game 12–15 s → 3,8 s + vòng nhìn 3D tròn (2026-10-17, commit e17bfcb) — nạp ô dưới chân trước, tô phòng nhớ vùng hở, chunk ngoài vòng hoãn"
metadata:
  node_type: memory
  type: project
  originSessionId: 365bdbc4-2cd1-4660-91e1-06151b153ca3
  modified: 2026-10-16T23:18:34.991Z
---

User yêu cầu 2026-10-17: thứ tự nạp lúc vào game (map + khối quanh người chơi trước, dân/quái/region xa sau, xa nạp chậm để khỏi khựng) + vùng vẽ chunk hình TRÒN thay vì vuông.

**Đo (LoadTrace "WildScene boot", file `persistentDataPath/Logs/load_trace.log`):** trước 12–15 s = `SectorWindow.Recenter` dựng đồng bộ 25 ô (5–6 s) + `WildHUDController.HandleGenerated` 4,3 s ← gần hết là `PaintRoomTintsInBounds` của ô giữa (ô có thành) 4,2 s.
**Sửa (e17bfcb):**
- `LocalMapBootstrap.startSyncRadius` (mặc định 0) + `SectorWindow.Recenter(cx, cy, syncRadius)`: chỉ ô dưới chân dựng đồng bộ; 24 ô còn lại `Update → Tick` luồng nền lấp dần (gần trước) — đã có sẵn cơ chế, chỉ đổi khởi tạo.
- `RoomDetector.FindEnclosedRoom(grid, seed, open)`: tràn ngưỡng ⇒ ghi mọi ô đã loang vào `open` (cùng vùng hở); tô phòng + tự lợp mái bỏ hạt giống trong `open`. Trước đó mỗi viên tường × 4 hướng loang lại ~400 ô.
- `VoxelGameView` vòng nhìn: `_viewRadius` (0 = (streamRadius+0,5)×S), `UpdateViewCircle` mỗi 0,25 s ẩn/hiện chunk; `SpawnChunkNow` hoãn chunk ngoài vòng (`SectorView.Deferred`, PendingSpawns giữ nguyên ⇒ `EndChunks` chỉ khi mọi chunk đã tạo). 5×5 ô: 1204/5625 chunk hoãn (21%).
- LoadTrace tách `OnGenerated` theo từng người nghe; log `[WildTilemapRenderer] vẽ đồng bộ — <bước>` khi ≥200 ms.
**Sau:** vào game 3,8 s (WorldGenerator 1,4 s, ô giữa 0,94 s, tô 2D ô giữa 0,26 s); 25/25 ô đủ sau vài giây, console sạch.
**Tầm theo chunk (commit sau e17bfcb):** `LocalMapBootstrap.viewDistanceChunks` (mặc định 32, đổi được lúc chơi) + `preloadMarginCells` 48 ⇒ `SectorWindow` chế độ vòng tròn (`LoadRadiusCells`, `SetFocus` theo VỊ TRÍ người chơi mỗi 0,1 s, gỡ khi > bán kính + đệm 16): ô world chạm vòng thì nạp. Kẹp `(streamRadius+0,5)·S − đệm` để không 2 ô chung khe vòng. Vòng vẽ 3D dùng cùng tầm ⇒ hết bị cắt phẳng ở ranh ô. Số ô: 8 chunk 4–9, 16 chunk 9–14, 32 chunk 24–32. User muốn tầm cao lúc tối ưu, máy yếu chỉnh thấp. RAM vẫn theo bậc ô world (dữ liệu về từng ô) — giữ dữ liệu theo từng chunk là việc lớn riêng (lưới vòng/region/Merge đánh khe theo ô).
**Chưa làm:** đo khựng lúc 24 ô nạp nền ngay sau khi vào; `WorldGenerator.Generate` 1,4 s vẫn đồng bộ; dân/quái chưa xếp ưu tiên theo khoảng cách lúc vào.

Liên quan: [[project-walk-hitch-2026-10-17]], [[project-chunk-commit]], [[project-load-trace]].

**Đo RAM theo ô (2026-10-17, Play Mode Editor, sau GC):** tầm 8 chunk (9 ô) heap managed 1794 MB; 32 chunk (25 ô) 2241 MB; về 8 lại 1804 MB (gỡ trả RAM đủ). ⇒ ~28 MB managed/ô world; phần còn lại (~1,55 GB) cố định (lưới vòng 6×6 khe cấp sẵn 1440² + Editor + data mod), KHÔNG giảm khi nạp ít ô. Unity native gần như không đổi (+60 MB). Kết luận: nạp theo chunk chỉ tiết kiệm ~28 MB/ô ⇒ chưa đáng viết lại lõi; nếu cần giảm RAM thì nhắm phần cố định trước.
