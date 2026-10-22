---
name: project-sim-render-split-profile
description: "Đo tách Game Loop ⟂ Render Loop (2026-10-17) — sim chỉ ~1,5 ms/khung, render thread ~4,4 ms là phần lớn; cú giật duy nhất còn lại = CityNpc slow tick A*/CanReach ~25 ms"
metadata:
  node_type: memory
  type: project
  originSessionId: 365bdbc4-2cd1-4660-91e1-06151b153ca3
  modified: 2026-10-17T13:19:11.914Z
---

User hỏi tách game loop khỏi render loop (2026-10-17). Đã chốt cách nói: mức 1 = gom sim về bước cố định + render nội suy (BƯỚC NỀN); mức 2 (luồng sim C# thuần + bản chụp) và mức 3 (DOTS) là HAI LỰA CHỌN thay nhau, áp từng hệ nặng, không phải cả game. Đo trước rồi mới quyết.

**Hiện trạng code:** `GameClock` đã là tick cố định 30 TPS (accumulator, `TickPhase` Ai→Job→Sim→World→Ui, ~38 file đăng ký tick) ⇒ mức 1 đã có một nửa. Nhưng DI CHUYỂN pawn/NPC/quái vẫn chạy `Update()` + `SimTime.DeltaTime` theo khung (cố ý, xem doc `TickRate`); 89 script có Update/LateUpdate (45 ở World/Wild, phần lớn UI). Tất cả chung luồng chính.

**Số đo (Editor, WildLocal 5×5, 3D, Profiler bật, chế độ nạp cũ):**
- Đứng yên 1500 khung: khung ~7,5 ms; PlayerLoop 3,2; script Update 1,5 (VoxelWorld 0,58, ContextActionController 0,16, còn lại < 0,1); **chờ render thread (`Gfx.WaitForGfxCommandsFromMainThread`) 4,6 ms** ⇒ đang nghẽn phía render, không phải sim.
- Đi bộ 97 s qua ranh ô world 363→364 (`WildPlayerToken.MoveTo`, +204 ô): p50 6,6 / p95 9,6 / p99 11,6 ms; 11 khung > 25 ms: 1 là khung MCP (1155 ms, loại), **8 là `CityNpcSystem.HandleSlowTick` → `BrainLoop` ~25 ms (`RP.DirectAStar` ~13 + `RP.Reach`/`RR.Bfs` ~10, đồng bộ)**. Qua ranh ô world KHÔNG còn khung > 25 ms.
- Bẫy: per-script `ProfilerRecorder` "X.Update() [Invoke]" không hợp lệ; phải đọc cây qua `ProfilerDriver.GetHierarchyFrameDataView` (tên đầy đủ "Nizhangjie.dll!...::X.Update() [Invoke]").

**User chốt hướng (2026-10-17):** lợi ích thật của luồng sim riêng là CÁCH LY (sim khựng không rớt khung), không phải thông lượng — nhưng tách cả sim = viết lại lõi (Transform/GameObject chỉ luồng chính). Làm từng phần: (1) đẩy phần nặng ra luồng nền; (2) quy ước mã mới = luật **B10** trong CODE_SYSTEM_RULE.md (commit 8515afdd + siết 31148dbf: sim không gọi thẳng UI, nợ di chuyển theo khung có tên, áp theo HÀM đang sửa; trỏ từ CLAUDE.md TL;DR + checklist phần E; test canh `Run Sim Purity Tests` commit 0b650fed — quét `*Ops.cs`/`JobGiver_*.cs`, mốc theo file, chỉ được hạ; commit e70f2c4e dọn 24→6: `Core/SimRandom` thay UnityEngine.Random, truyền LocalMapData thay Find; 6 chỗ còn lại cần đổi cấu trúc — CompanionRegistry là MonoBehaviour scene (CompanionFarmOps/CreatureCareOps), WeaponProcOps hất văng chưa nhận lưới, CompanionOps đọc biome qua bootstrap, BlockPlacementOps.WallAt dùng Time.frameCount làm bộ đệm; CHƯA verify Play Mode lần dọn này); (3) sau này đổi chỗ chạy.

**Bước 1 XONG — commit 77ca34fb:** `CityNpcSystem.BeginIndoorWalk` (dân trong nhà đang mở nóc đổi chỗ) gọi `FindPath` đồng bộ = thủ phạm 25 ms. Nay `PrepareQuery(deferCorridor)` → `PumpIndoorPaths` (Update, pha 1 ngân sách 1,5 ms) → A* trên `Task.Run` → giao nếu vẫn mới nhất (`_latestIndoorReq`, thế hệ, còn Revealed, IndoorWalkTo không đổi); chờ thì đứng yên. Đo 2D đi bộ 363→364: SlowTick max 7 ms, BrainLoop 4,6, Pump 4,0; đứng yên 90 s: BrainLoop ≤1,95, Pump ≤1,54; RP.DirectAStar/RP.Reach trên luồng chính = 0. Test Npc 43/0, NpcBrain 39/0. ⚠ Lần đo bật 3D bằng `TurnOn()` qua reflection: máy chậm chung (p50 19 ms) và avatar KHÔNG đi dù MoveTo trả true — chưa truy, không liên quan thay đổi này. Lần đo 2D có `WildTilemapRenderer.PaintQueueRoutine` ăn nhiều (chỉ chạy khi 3D không giữ camera) — so sánh tổng khung phải cùng chế độ xem.

**Kết luận:** đẩy cả sim ra luồng riêng gần như không lợi ích (sim 1,5 ms). Việc đáng làm: (1) A*/CanReach của nhịp chậm dân thành chạy nền hoặc chia ngân sách (giống quái 7da03a8e); (2) xem render thread 4,4 ms. Liên quan [[project-walk-hitch-2026-10-17]], [[project-canreach-findpath-gap]], [[project-tick-spread]], [[project-fps-profile-2026-10-14]].
