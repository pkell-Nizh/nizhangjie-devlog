---
name: project-walk-hitch-2026-10-17
description: "Giật khi đi bộ (2026-10-17) — không phải mod; thủ phạm lớn là giờ đi làm của dân thành (Version đổi liên tục vứt cache) — đã sửa; còn CanReach BFS"
metadata:
  node_type: memory
  type: project
  originSessionId: 365bdbc4-2cd1-4660-91e1-06151b153ca3
  modified: 2026-10-17T16:38:15.677Z
---

User báo "đi bộ vẫn khựng". Đo 3D trong Editor ngay sau khi vào game: p50 22,6 / p99 77 ms.
**Không phải nạp data mod:** 123 lượt `[JsonDataLoader]` đều thuộc `LocalMapBootstrap.Start` (~17 s); lúc chơi 0 lượt (marker `Data.LoadFile`/`Data.LoadFolder` mới gắn vào JsonDataLoader).
**Đã sửa:** `CreatureTeamPanel` phá + dựng lại cả UI mỗi 0,5 s (Canvas 50–280 ms) ⇒ chỉ dựng lại khi chữ ký nội dung đổi; `BlockTicks` nhịp ngẫu nhiên tới 37 ms/khung ⇒ trần `maxMsPerFrame` 1,5 (JSON block_ticks, cả 2 thư mục), mẫu thừa giữ làm nợ. Sau sửa p50 16,5, LateUpdate 3D 5,5→2,1 ms.
**Giật giờ đi làm của dân thành (đã sửa, 5b97ea6 + commit sau):** thật ra không phải do load mà do GIỜ ĐI LÀM (tái hiện: `GameClock.SetSpeed(3)` từ 0h tới 11h, bộ ghi `ProfilerRecorder` trong `EditorApplication.update`). Gốc: `Regions.Version` tăng ~2 lần/giây vì bất kỳ ô nào trong 5×5 ⇒ kết quả đường nền bị vứt + cache đường đi làm/RegionCost bị xoá ⇒ tính corridor lại mãi (270–380 ms/chuyến). Sửa: kiểm từng ô đường (`PathStillWalkable`) thay vì so version; `PathRegion.CostCache` trên object; `RegionPathfinder.CorridorSearch` chạy tiếp được, `CityNpcSystem.StepCorridorWork` 2 ms/khung. Sau: DrainPathQueue tệ nhất 274→11,7 ms, khung ≥8 ms 131→20.
**Tới-được dựng dần (commit sau 517ee79):** `RegionReachability.ReachSearch` (BFS hai chiều chạy tiếp được); `PrepareQuery(deferCorridor)` loang dần rồi mới corridor; CanReach đồng bộ chỉ còn cho chuyến ngoại ô. Giờ đi làm ×3: DrainPathQueue tệ nhất 6,4 ms.
**Quái không chặn (commit sau):** `BeastBehaviour.RequestPath` → `PrepareQuery(deferCorridor)`, `PumpPaths` 2 ms/khung, A* cấp ô trên ThreadPool, chỉ giao yêu cầu mới nhất; quái giữ đường cũ khi chờ. `Beast.PathStep` max 2,1 ms (trước 20–35).
**Đo lại tối 2026-10-17 (f4fd5e2e):** `Beast.PathStep`/`RR.Bfs` 6,7 ms trong Profiler là do Profiler thổi phồng; log tạm cho thấy 110 s đi bộ chỉ 6 lần Step vượt ngân sách 0,6–1,6 ms (64 lượt mới xem đồng hồ) ⇒ nay mỗi 16 lượt (cả CorridorSearch). `EnsureRebuilt` KHÔNG chạy trong BFS (marker `PRG.EnsureRebuilt` = 0). Cú 32 ms lúc mới vào = JIT lần đầu. Dân thành trong nhà đã chạy nền (77ca34fb). **22 chỗ `CanReach` đồng bộ** còn lại hầu hết trong `JobGiver_*` của pawn (Haul/Construction/Eat/Rest/Tend/Craft/MentalBreak/Threat…) + IdleActivityPipeline/WorkAssigner/BlockWork/AmbientSpotScan; sửa = đổi hợp đồng JobGiver (việc lớn, chưa làm, chờ user).
**Đo 22 chỗ CanReach đồng bộ (2026-10-17, WildLocal ×3, ~150 s, 4 pawn):** log tạm theo nơi gọi (StackTrace) chỉ khi phải loang BFS. Pawn `JobGiver_*` = **0 lần loang** (toàn trúng đường nhanh/cache); IdleActivityPipeline 7 lần, AmbientSpotScan 2 lần, đều <0,02 ms. Chỗ đắt duy nhất: `CityNpcSystem.DrainPathQueue` (chuyến ngoại ô, `CityNpc.Q.CanReach`) 34 lần, max **11,9 ms** (1 lần), 3,1 ms (1 lần); `LeadOutPath` 180 lần, max 1,2 ms. ⇒ KHÔNG cần đổi hợp đồng JobGiver; nếu sửa thì chỉ chuyến ngoại ô dùng ReachSearch dựng dần. Giới hạn: ít pawn, không có việc khuân/xây lớn. Khung >50 ms (61/13938) không do CanReach — chưa truy.
**Còn lại:** vài lần `RR.Bfs` 5–11 ms từ hệ khác gọi `CanReach` đồng bộ (pawn AI?) — chưa truy; cache `RegionReachability` vẫn xoá theo Version.
Marker mới: `VGV.L.*` (từng bước VoxelGameView.LateUpdate), `RR.Resolve`/`RR.Bfs`.

Liên quan: [[project-chunk-commit]], [[project-tick-spread]].
