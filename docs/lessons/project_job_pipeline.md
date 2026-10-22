---
name: project-job-pipeline
description: "M16 Job Pipeline (JobDriver/Toil kiểu RimWorld) — nền tảng thực thi job pawn, ĐÃ CODE 2026-07-16 (đêm, user ngủ), Construction đã migrate, CHƯA verify Unity"
metadata: 
  node_type: memory
  type: project
  originSessionId: a2f2bd3e-a656-4566-b430-297ea25134c4
  modified: 2026-07-28T12:02:45.036Z
---

User yêu cầu (trước khi đi ngủ, giao tự cân nhắc): "tham khảo RimWorld tạo pipeline làm việc thật tốt +
code nền tảng để thoải mái thay các core code cũ". Bối cảnh thúc đẩy: TRONG 1 NGÀY test Play Mode
(2026-07-16), `JobGiver_Construction` bản closure dính 4 bug thật cùng lớp "tự quản vòng đời bằng tay"
(xem [[project_wall_building]]) + 2 lớp bug cũ hơn (unsubscribe sai delegate 2026-07-12, callback tới
muộn sau cancel — lỗ hổng chưa kịp nổ).

**ĐÃ CODE 2026-07-16 (đêm) — M16 Job Pipeline, mirror RimWorld Job/JobDriver/Toil:**
- `World/Wild/Ai/Jobs/JobDriver.cs` — `Toil` (3 kiểu: tức thời / WaitsForSignal chờ callback / Tick) +
  `JobDriver` (chuỗi toil). 5 bảo đảm: (1) `OnCleanup` đúng 1 lần MỌI đường kết thúc; (2) `FailWhen` khai
  1 lần, check trước mỗi bước + mỗi tick; (3) KHÔNG subscribe GameClock — `PawnAiBrain` bơm
  `PawnJob.Tick` 1Hz (field mới), TRƯỚC gate IsAvailableForWork (job dở vẫn tiến độ khi pawn vào party —
  giữ hành vi cũ); (4) sau `Ended` mọi callback/tick muộn = no-op + `Token.StopMoving()` (method mới trên
  WildPlayerToken); (5) `Cancel` KHÔNG gọi onComplete (hợp đồng brain cũ).
- `World/Wild/Ai/Jobs/Toils.cs` — factory: `Goto` (MoveTo, fail nếu không đường; caller resolve ô đứng
  qua WorkPositioning TRƯỚC), `Instant` (action 1 lần, muốn lý do fail riêng thì `d.Fail(reason)` rồi trả
  false), `WorkUntil` (tick 1Hz, tiến độ lưu TRÊN OBJECT ĐÍCH — quy ước để job nhả giữa chừng không mất công).
- `JobGiver_Construction` ĐÃ MIGRATE làm mẫu chuẩn — GIỮ NGUYÊN 100% hành vi chống-bug (check túi thật,
  đứng ô kề né blueprint, gate hoàn thành + đẩy pawn đứng lì + timeout 15s nhả việc), bỏ toàn bộ machinery
  GameClock/StopTicking/Bail. `TryGiveJob` selection KHÔNG đổi (test cũ vẫn phải pass).
- Test MỚI: `Tests/Editor/JobDriverTest.cs` — menu `Run JobDriver Tests`, 9 case test đúng 5 bảo đảm
  (Token=null, không Play Mode).
- Doc: `Assets/Scripts/Nizhangjie/M16_JOB_PIPELINE_DESIGN.md` (mapping RimWorld→Nizhangjie, bảng
  migration, gotchas). CLAUDE.md §5 đã thêm bullet M16.

**Quyết định tự cân nhắc (user ngủ, cần biết khi dậy):**
1. KHÔNG migrate `JobGiver_Designation`/`WildGatherSystem` đêm nay — `Begin/Complete` phục vụ CẢ đường
   player lệnh tay LẪN AI; đụng `_busy` (nguồn "bận" chung 2 đường, fix bug #3 2026-07-12) khi chưa verify
   được là rủi ro. Hướng migrate đã ghi trong doc §5.
2. KHÔNG dựng ReservationManager riêng — giữ `Claimed` trên object đích (Designation/BuildPiece), đúng
   quyết định M15 "object tự mang state".
3. `PawnJob` giữ nguyên hợp đồng cũ (Execute/Cancel) + thêm `Tick` optional — JobGiver kiểu cũ chạy y
   nguyên, không breaking.

**CHƯA VERIFY UNITY (việc sáng dậy — ghi cả trong doc §7):** recompile → chạy `Run JobDriver Tests` +
`Run JobGiverConstruction Tests` + `Run PawnAiBrain Tests` + `Run WorkAssigner Tests` → Play Mode lặp lại
kịch bản hôm trước (blueprint + kho + pawn ôm liệu sẵn + pawn đứng lì trên ô xây). Log `[WildBuild]` giữ
format cũ; fail giờ log prefix `[PawnJob]`.

Gotcha: namespace `World.Wild.Ai.Jobs` ≠ `LocalMap.Jobs` (job kiếm tiền city — hệ khác hẳn, ngoài phạm vi).

**§Hauling split — ĐÃ CODE 2026-07-17 (cùng ngày, sau khi 4 test suite pass + ảnh Play Mode lộ thêm 1 bug
"xây khối đặc tự nhốt ruột"):** user hỏi so sánh RimWorld ("AI ta thua RimWorld rõ") rồi bảo "làm luôn" —
2 việc gộp 1 lần:

1. **Bug #5 (Exposedness ordering)** — kéo khối tường ĐẶC (vd 5x5) → chọn "gần pawn nhất" trước luôn ưu
   tiên vòng ngoài (builder đứng ngoài) → vòng ngoài Built (chặn ô) TRƯỚC KHI ruột kịp xây → ruột (3x3
   giữa) bị vây kín vĩnh viễn. Fix: `JobGiver_Construction.Exposedness(data, pos)` (mới, static) đếm ô kề
   còn "mở" thật (không phải Wall Blueprint/Built, không phải địa hình chặn) — Pass 1 ưu tiên Exposedness
   THẤP NHẤT trước (bị vây nhiều nhất), dist chỉ tie-break → tự nhiên xây "từ ruột ra ngoài" mirror
   RimWorld, không cần biết trước khối đặc hay khung rỗng (khung rỗng thì mọi exposedness ~bằng nhau, hành
   vi y hệt cũ). Door không tính là "chặn" trong Exposedness.

2. **Tách Hauling ⟂ Construction** (câu hỏi thứ 2 user đặt ra, RimWorld tách 2 WorkType độc lập) —
   `JobGiver_Haul` (mới, `World/Wild/Ai/JobGivers/JobGiver_Haul.cs`, Tier 3, DÙNG CHUNG capability
   `core:building` — mô tả JSON `capabilities/building.json` vốn đã "khuân + dựng") tìm blueprint còn
   thiếu liệu gần nhất+tới được, lấy từ túi pawn TRƯỚC (mirror bug#1 fix cũ, giờ dời qua đây), thiếu thì
   kho gần nhất CÒN HÀNG (`FindNearestStorageWithSome` — không cần đủ hết 1 kho, nhiều hauler/chuyến gộp
   dần), giao xong KHÔNG đứng lại xây. `JobGiver_Construction` bỏ HẲN phần khuân (Goto kho/Instant lấy
   liệu/carryingAlready) — Pass 1 giờ có thêm gate `MaterialsDelivered >= costAmount` mới cho qua.
   `BuildPiece.ReservedForHaul` (int, field mới) = số liệu hauler KHÁC đã reserve nhưng chưa giao — trừ
   khỏi "còn thiếu" TRƯỚC khi 1 hauler mới xét, chặn double-book; cộng lúc `Execute` (mirror
   `best.Claimed=true` — mutate ở Execute không phải TryGiveJob), trừ lúc giao xong HOẶC lúc fail/cancel
   (flag `delivered` trong closure tránh trừ đôi khi cleanup chạy SAU khi đã giao). `PawnAiBrain.Awake`
   đăng ký Construction TRƯỚC Haul (ưu tiên hoàn thành piece sẵn sàng hơn bắt đầu chuyến khuân mới).

**Test:** `JobGiverConstructionTest.cs` SỬA LẠI hoàn toàn (bỏ hết Storage/GlobalWarehouse — Construction
không còn đụng gì tới đó; test 3/4/5 đổi ý nghĩa: giờ test qua `MaterialsDelivered` trực tiếp thay vì mô
phỏng kho). `JobGiverHaulTest.cs` (mới, 8 case) — capability gate, free-piece skip, đã-đủ-liệu skip,
không nguồn nào→null, túi pawn đủ→job không cần kho, kho đủ+reachable→job, kho unreachable→null,
ReservedForHaul đã che hết→null (chống double-book). CHƯA chạy lại 2 bộ này lẫn `Run JobDriver Tests`
sau đợt sửa này — việc đầu tiên cần làm khi quay lại.

Liên quan: [[project_wall_building]] (context đầy đủ 5 bug + ảnh Play Mode).

Liên quan: [[project_wall_building]] (4 bug động lực), [[project_pawn_ai_brain]] (brain/tier),
[[project_work_positioning]] (ô đứng thao tác — bạn đồng hành của pipeline này).

**Bug #6 (2026-07-28) — `JobGiver_Designation` thiếu `PawnJob.Cancel` (đúng cảnh báo "CHƯA migrate" ở
trên, giờ mới nổ):** user báo pawn đơ giữa đường tới job (đi ngang bị 1 job tier cao hơn interrupt) rồi
mọi lệnh sau (Move/Store/Take...) đều báo "đang bận" vĩnh viễn. Root cause: `JobGiver_Designation` là
JobGiver DUY NHẤT trong 9 cái không set `Cancel` (Construction/Haul/BuildStructure/HaulStructure/
MeleeAttack/Threat/IdleActivityPipeline đều có `Cancel = _ => driver?.Cancel()`) — job của nó chạy qua
`WildGatherSystem.Begin` (busy-tracking RIÊNG, `_busy` HashSet, NGOÀI `PawnAiBrain._active`, cố ý — xem
quyết định #1 ở trên). Khi brain interrupt (`runningJob.Cancel?.Invoke(token)` = no-op vì null) rồi job
mới gọi `Token.MoveTo` cho đích riêng nó → `WildPlayerToken.MoveTo` LUÔN reset `_path/_onArrive` bất kể ai
gọi (không check "ai đang sở hữu path hiện tại") → callback `WildGatherSystem.Complete` (nơi DUY NHẤT gỡ
`_busy`) không bao giờ bắn → token kẹt `_busy` vĩnh viễn → `ForceJob`/`IsBusy` sau đó luôn false.
Fix: thêm `WildGatherSystem.Cancel(token)` (gỡ `_busy` + `StopMoving()`) + wire vào
`JobGiver_Designation`'s `Cancel` (nhả `designation.Claimed` luôn, mirror nhánh Begin-fail).
**Bẫy chung cần nhớ:** BẤT KỲ `PawnJob` nào dispatch qua 1 hệ "busy" RIÊNG ngoài `_active` (không chỉ
Designation — coi chừng nếu thêm hệ tương tự sau này) BẮT BUỘC phải có `Cancel` gỡ đúng state hệ đó, vì
`WildPlayerToken.MoveTo`/`StopMoving` không biết/không quan tâm ai đang "sở hữu" path hiện tại.

**Bug #6b (cùng ngày, user báo "vẫn lỗi" sau fix #6 — đường THỨ HAI hở, cùng họ):**
`WildSceneInteraction.HandleAction` case `InteractableKind.Feature` (verb hái/chặt/đào TRỰC TIẾP, KHÁC
Designate — actor = pawn đang chọn qua info panel) gọi THẲNG `_gather.Begin(...)`, KHÔNG qua
`PawnAiBrain.ForceJob` — verb DUY NHẤT trong `HandleAction` làm vậy (Move/Store/Take/PickUp/Attack đều qua
`ForceJob` Tier 0). Nếu pawn đang chọn hiện có job idle (Wander/StandingOrder Tier 4/5) chạy trong
`_active`: `Begin` cứ gọi `MoveTo` đè lên (job idle không biết, không bị Cancel) → job idle mồ côi vẫn
nằm `_active` → SlowTick sau tìm được job khác interrupt nó → Cancel job mồ côi gọi `Token.StopMoving()`
→ xoá NGANG path/callback của lệnh gather ĐANG THẬT chạy → `_busy` kẹt vĩnh viễn (y hệt bug #6, đường
khác). Fix: `WildSceneInteraction.HandleGatherAction` (mới, thay chỗ gọi `_gather.Begin` trực tiếp) — bọc
qua `PawnAiBrain.ForceJob` (Tier 0, `Cancel = token => _gather.Cancel(token)`), mirror hệt
`HandleStorageAction`/`HandleMove`. **Bài học:** khi thấy 1 verb KHÔNG theo pattern chung (ở đây: mọi verb
khác đều qua `ForceJob`, riêng verb này gọi thẳng subsystem) — đó chính là chỗ nghi trước tiên, không chỉ
sửa 1 JobGiver rồi coi là xong.
