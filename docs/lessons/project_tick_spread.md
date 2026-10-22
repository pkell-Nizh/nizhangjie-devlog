---
name: project-tick-spread
description: "Hạ tầng phân rã tick (2026-10-15) — TickSpread (pha theo khoá, mỗi tác nhân 1 lần/chu kỳ) + BudgetedQueue (gộp theo khoá, ngân sách ms/khung); đã cắm PawnAiBrain, BeastBehaviour, VoxelGameView chunk rebuild; commit 884f16c"
metadata:
  node_type: memory
  type: project
  originSessionId: d93f285a-5ead-48f8-9bcd-66fa9efac21c
  modified: 2026-10-15T14:22:33.375Z
---

User chốt 2026-10-15: "hạ tầng trước, quái sau" — dựng hệ phân rã tick dùng chung, cắm vào chỗ đang giật; VIẾT LẠI AI quái (3D, nhiều tầng, KinematicBody) là bước RIÊNG sau, xây trên hạ tầng này. User đồng ý `BeastBehaviour` đã lỗi thời (2D, `surfaceOnly`, `System.Random` không seed, `FindFirstObjectByType` mỗi đòn cắn).

**Mảnh 1 — `Core/Time/TickSpread.cs`:** pha = FNV-1a(khoá chuỗi) hoặc trộn bit (khoá số) mod chu kỳ; tác nhân chạy khi `(tick + pha) % chu kỳ == 0`. Theo TICK của GameClock ⇒ tất định, pause là đứng. Khoá phải là id dữ liệu (PawnRef, `actor.instanceId`), KHÔNG `GetInstanceID`/`string.GetHashCode`.

**Mảnh 2 — `Core/Time/BudgetedQueue<TKey>`:** xếp việc theo khoá (khoá trùng ⇒ giữ chỗ cũ, việc mới nhất thắng), `Drain(budgetMs, minItems=1)` FIFO tới hết ngân sách. Có số lượt xếp: Remove rồi xếp lại ⇒ xuống CUỐI (bug test bắt được ở bản đầu). Không runner toàn cục — chủ hàng tự xả.

**Đã cắm:**
- `PawnAiBrain`: `Subscribe(TickPhase.Ai)` mỗi tick, chỉ pawn tới lượt (chu kỳ 30 tick = 1 s như OnSlowTick cũ), pha nhớ trong dict theo PawnBody.
- `BeastBehaviour`: lượt nghĩ theo tick (`_thinkTicks` 34, pha theo instanceId), `thinkNow` (đường đứt) tối đa `_maxUrgentPerTick` 6 con/tick; bỏ `thinkTimer`/Rand; di chuyển vẫn mỗi khung. BeastSpawner bỏ dòng random timer.
- `VoxelGameView`: `OnChunkChanged` xếp `_chunkQueue` (khoá (sector, chunk)), xả đầu `LateUpdate` cùng khung, 4 ms/khung; `UpdateLightRebuilds` gọi `RebuildChunkNow` và bỏ chunk đã có trong hàng. Chữa luôn giật DoorTrafficSystem (mỗi cửa mở/đóng từng = 1 lần dựng chunk tức thì).

**Đo sau (WildLocal 5×5, Editor, đứng yên 40 s):** p50 4,0 / p95 6,8 / p99 12,6 ms, 8 khung >25 ms; `PawnAiBrain` max 0,22 ms/tick (trước 23–43); `BeastBehaviour.OnTick` avg 0,14 max 12; 8 lần đập khối cùng chunk ⇒ 1 lần dựng, mỗi lần đập ~1 ms (trước 5–13). Test `Run Tick Spread Tests` 18/18; PawnAiBrain 4, Corpse 19, Loot 30, WildCombat 53, WorkAssigner 16 xanh.

**Dựng region DẦN (cùng ngày, xác nhận + sửa):** đo thật: dựng region 1 ô world = 80 ms (`Pathfinder`) + 41 ms (`RoadPathfinder`), mỗi bộ ~175k region trên 5×5 (section 10 ô), dồn vào câu hỏi đường đầu tiên. `PathRegionGrid.DeferBulk` (bật bởi `LocalMapBootstrap.FlushDeferredRegions` mỗi khung, 2 ms/bộ, khối gần avatar trước): khối mới nạp + vùng đổi độ cao ≥ 64 section vào sổ chờ (region gỡ, ô = -1 "chưa sẵn sàng"), câu hỏi KHÔNG ép dựng; sửa lẻ vẫn dựng ngay; dựng lần đầu lúc vào game như cũ. Test `Run Ring Grid Tests` 44 (+11 Defer_*). Play Mode: qua ranh, sổ chờ lên 400/ô rồi tới ~1800 (độ cao đẩy vào sau ⇒ mỗi ô dựng 2 lần — phẳng rồi có độ cao; có thể bỏ lượt phẳng), xả dần, không còn khung nào do region.
**Qua ranh vẫn còn (chưa sửa):** ghép ô `LocalMapBootstrap.Update` 96–110 ms/ô + gỡ ô 52 ms (`RemoveSector` 34) trên luồng chính; `BeastBehaviour.OnTick` avg 1,5 max 31 ms/tick lúc qua ranh (nghi nghĩ-gấp × tìm đường khi địa hình đổi liên tục); `VoxelWorld/VoxelGameView` dựng ô 15–25 ms. Tua ×3: random tick khối + dân + tô 2D ⇒ p50 23 ms. `RegionReachability` BFS câu trả lời "không tới được" duyệt cả đồ thị 175k region (~8 ms) và cache xoá mỗi Version — nên dùng nhãn thành phần liên thông.
**Bẫy đo:** phiên remote desktop của user ⇒ chờ GPU 5–33 ms/khung, so FPS tuyệt đối với số cũ là sai. Avatar ở Classic bị AI giao việc rảnh (ngắm cảnh) đè `MoveTo` thô ⇒ đi thử phải qua `PawnAiBrain.ForceJob` (Tier 0, dựng PawnJob bằng Expression). `FindPath` có tham số tuỳ chọn ⇒ tra reflection đúng overload, đừng kết luận "path null".
**Còn lại khác:** `CityNpcSystem.HandleSlowTick` max 19 ms; giờ in-game: `GameCalendar.HandleHour` 11 ms + `CityNpcSystem.HandleHourTick` 7 ms; lớp 2D vẫn tô tilemap khi xem 3D (~6 ms/khung lúc nạp ô); `JobGiver_HaulToStorage` quét 686 kho khi đồ rơi không dùng được.

**Bẫy gặp lúc làm:** sửa file `.unity` bằng sed khi Unity đang mở ⇒ hộp thoại "modified externally" treo Unity (MCP mất, CPU đứng); gỡ được bằng PowerShell `EnumWindows` + `SendMessage(WM_COMMAND, id nút)` lên hộp thoại `#32770`. Lệnh `sleep` foreground bị chặn ⇒ dùng Bash `run_in_background`.

Liên quan: [[project_hitch_profile_2026-10-15]], [[project-fps-profile-2026-10-14]], [[project_pawn_ai_brain]].
