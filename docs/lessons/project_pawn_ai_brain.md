---
name: project-pawn-ai-brain
description: "M13 Pawn AI Brain (ThinkTree/JobGiver) — design+code 2026-07-11, verify hoạt động + Orders panel RimWorld click/drag designate 2026-07-12"
metadata: 
  node_type: memory
  type: project
  originSessionId: 6e856f28-92f0-457f-bb66-798f17027e73
  modified: 2026-07-25T17:36:02.828Z
---

2026-07-11: thiết kế + CODE xong cùng ngày tầng "AI brain" bao trùm cho pawn trên Wild map, nối tiếp
[[project-pawn-system-masterplan]]. Doc đầy đủ: `Assets/Scripts/Nizhangjie/M13_PAWN_AI_BRAIN_DESIGN.md`.

**ĐÃ CODE (ngay sau khi chốt thiết kế):** `World/Wild/Ai/` (mới) — `PawnAiContext.cs` (struct),
`IPawnJobGiver.cs`, `PawnJob.cs`, `PawnAiBrain.cs` (đổi tên từ `World/Wild/Work/PawnWorkScheduler.cs`, GIỮ
NGUYÊN GUID .meta nên wiring `WildLocal.unity` không đứt — di chuyển bằng `mv` cả .cs lẫn .cs.meta, KHÔNG
dùng Unity Editor rename), `JobGivers/JobGiver_Designation.cs` (Tier 3, bọc nguyên
`WorkAssigner.FindBestAssignment`, thêm guard `_gatherSystem == null → return null` mà bản gốc có nhưng
generalize loop đã bỏ). Field Inspector `_bootstrap/_gatherSystem/_renderer` giữ nguyên tên → Unity tự
re-resolve, KHÔNG cần wire lại tay. Test: `Tests/Editor/PawnAiBrainTest.cs` (menu "Run PawnAiBrain Tests", 4
case — chỉ test JobGiver_Designation plain-C#, KHÔNG test PawnAiBrain MonoBehaviour orchestrator vì cần Play
Mode thật). Đã sửa comment tham chiếu tên cũ ở 6 file khác (WorkAssigner.cs, WildGatherSystem.cs,
WildSceneInteraction.cs, WildTilemapRenderer.cs, Party.cs, WorkAssignerTest.cs) — thuần doc-comment, không
đổi logic.

**ĐÃ VERIFY 2026-07-12 — pawn tự đi làm THẬT trong Unity, user xác nhận "pawn tự đi làm rồi".** Compile OK,
2 bộ test pass. Quá trình debug lộ ra 2 bug/gap kiến trúc THẬT (không liên quan refactor tên, đã sửa):

1. **`WildLocal.unity` không có nguồn `GameClock` nào** — `GameBootstrap` (tạo GameClock) chỉ sống trong
   `Bootstrap.unity`, scene này KHÔNG nằm trong `EditorBuildSettings` (không tự chạy). `CityScene.unity` có
   sẵn `LocalMapClock` (fallback tự tạo GameClock nếu thiếu — xem code cũ), nhưng `WildLocal.unity` thì
   KHÔNG — gap có từ trước, không ai để ý vì chưa test tay flow AI thật bao giờ. Fix: user tự thêm GameObject
   gắn `LocalMapClock` (`Nizhangjie.LocalMap.Time`) vào `WildLocal.unity`.
2. **`LocalMapClock` fallback tạo `GameClock` KHÔNG `DontDestroyOnLoad`** — nếu tạo ở scene trước (vd
   WorldMap) rồi chuyển sang WildLocal, GameClock chết theo scene cũ. Đã sửa `LocalMap/Time/LocalMapClock.cs`
   Awake(): GameClock giờ tạo trên **GameObject RIÊNG + `DontDestroyOnLoad`** (không còn attach lên GameObject
   của chính LocalMapClock nữa), sống xuyên mọi scene sau đó — chỉ cần 1 LocalMapClock chạy ở BẤT KỲ scene
   nào trong phiên chơi là đủ cho toàn bộ session.
3. **`PawnAiBrain.OnEnable()` chỉ thử tìm GameClock ĐÚNG 1 LẦN** — nếu GameClock chưa tồn tại đúng lúc đó
   (vd do thêm LocalMapClock giữa chừng lúc đang Play, hoặc thứ tự Awake/OnEnable giữa 2 GameObject không
   như kỳ vọng), `OnSlowTick` không bao giờ được subscribe, không tự phục hồi. Đã sửa: thêm `Update()` tự
   thử `TryResolveClock()` lại MỖI FRAME cho tới khi tìm được (rẻ, chỉ 1 null-check/frame sau khi đã tìm
   thấy) — hết phụ thuộc thứ tự khởi tạo chính xác giữa các GameObject.

Debug logs tạm thời (`[PawnAiBrain] tick:...`, `[JobGiver_Designation] LogWhyNoMatch`, error đỏ ở OnEnable)
ĐÃ DỌN SẠCH sau khi xác nhận fix đúng — code hiện tại KHÔNG còn log debug thừa.

**Bài học cho lần sau:** khi thêm MonoBehaviour mới phụ thuộc `GameClock`/service qua `ServiceLocator` ở
Wild scene, ĐỪNG giả định service đã sẵn sàng lúc `OnEnable()` — Wild scene không có `GameBootstrap`/
`LocalMapManager` lo việc này như City, nên cần tự thủ (lazy-retry qua `Update()` là pattern an toàn nhất,
xem `PawnAiBrain.TryResolveClock`).

**Mô hình:** RimWorld ThinkTree/JobGiver — priority list cố định (KHÔNG Utility AI chấm điểm, khớp quyết định
cũ "không làm Needs/Mood"). Contract: `IPawnJobGiver.TryGiveJob(PawnAiContext) → PawnJob?`, duyệt tier 1→5 từ
trên xuống, job đầu tiên khớp thì làm.

**5 tier đã chốt (Party > Threat > Designated Work > Standing Order > Wander):**
1. Party/Player control — gate có sẵn (`Disciple.IsInParty`), không phải JobGiver thật, brain skip pawn đó.
2. Threat reaction — GIỮ CHỖ, chưa code (combat/aggro beast còn stub, xem masterplan).
3. Designated Work — ĐÃ CÓ, chỉ bọc lại: `JobGiver_Designation` gọi nguyên `WorkAssigner.FindBestAssignment`.
4. Standing Order — GIỮ CHỖ (vd tu luyện khi rảnh), nội dung chưa thiết kế.
5. Wander — GIỮ CHỖ (fallback cuối), thuật toán chưa quyết định.

**Migration đã chốt:** `PawnWorkScheduler` → đổi tên/generalize thành `PawnAiBrain` (KHÔNG viết song song class
mới) — vẫn 1 MonoBehaviour tự wire `GameClock.OnSlowTick`, vẫn `FindObjectsByType<WildPlayerToken>` (chưa cần
registry pawn). `_busy: HashSet` → `_active: Dictionary<WildPlayerToken, ActiveJob>` (track job nào, không chỉ
bận/rảnh) để hỗ trợ interrupt.

**Interrupt:** pawn rảnh → duyệt full 5 tier; pawn bận → mỗi tick chỉ duyệt tier CAO HƠN tier đang chạy (rẻ).
Cần thêm Cancel API cho `WildGatherSystem`/`WildPlayerToken.MoveTo` (hiện chỉ có callback-khi-xong, không hủy
giữa chừng) — nhưng chỉ cần làm khi Tier 2 có nội dung thật, chưa cần ngay vì hiện chỉ Tier 3 hoạt động.

**Ngoài phạm vi (chốt rõ, để sau):** nội dung Threat/Standing Order/Wander, Cancel API, Build/Haul/Hunt
WorkGiver mới, persist ActiveJob qua save/load.

**ĐÃ CODE 2026-07-12 (cùng phiên, sau khi verify pawn tự đi làm) — UI Orders panel THẬT, DATA-DRIVEN (không
phải 1 nút cứng — user sửa lại giữa chừng, xem bên dưới), lấp gap đã ghi trong masterplan ("Orders panel vẫn
NotImplementedYet"):**

Bản nháp ĐẦU (1 nút "Đánh Dấu" duy nhất, auto-suy loại việc qua `WorkGiverDef.FindForCategory`) — **user bác
bỏ giữa chừng**: mục tiêu thật là Orders (và Zone/panel khác sau này) phải theo pattern data-driven như
`WildStructurePanel` đã làm (đọc JSON → grid tự sinh nút → modder thêm JSON là có nút mới), KHÔNG phải 1 nút
tay. Đã sửa lại đúng hướng:

- `WildToolMode` (`World/Wild/UI/WildSceneInteraction.cs`) +giá trị `Designate` (append). Dùng CHUNG cơ chế
  drag-rect có sẵn của Zone tool (`_dragging`/`HighlightRect`/Esc-huỷ) — KHÔNG viết lại.
- `WorkGiverDef.cs` +field `iconAddress` (mirror `WildBuildingDefinition.iconAddress`, rỗng = fallback glyph).
- `WildActionButtonSlot.cs` (mới, `World/Wild/UI/`) — prefab component RIÊNG cho nút taskbar (icon/glyph
  fallback/label/nền/click/SetSelected), **KHÔNG dùng chung `ItemSlot`/`InventorySlotUI`** (user chốt rõ:
  muốn prefab riêng cho taskbar, không tái dùng ô inventory — khác quyết định mặc định "tái dùng" thường
  thấy trong dự án, lần này CỐ Ý tách vì style/kích thước taskbar khác). Dùng lại được cho Zone/panel khác
  khi chuyển data-driven sau, không cần tạo thêm prefab.
- `WildOrdersPanel.cs` (mới, mirror `WildStructurePanel.cs`) — grid TẤT CẢ `WorkGiverDef` (`DataRegistry<WorkGiverDef>.GetAll()`,
  `LoadFromFolder("work_givers")`), mỗi nút Instantiate+Bind (icon/glyph/label = displayName). Click 1 nút →
  `WildSceneInteraction.SelectDesignateTool(workGiverId)` (đổi signature, giờ nhận id cụ thể) + `SetSelected`
  sáng viền nút đang chọn.
- `WildSceneInteraction.SelectDesignateTool(string workGiverId)` (đổi từ parameterless) lưu
  `_designateWorkGiverId`. `CommitDesignateDrag` giờ CHỈ đánh dấu feature khớp ĐÚNG WorkGiverDef đã chọn
  (`workGiver.MatchesCategory`), KHÔNG còn auto-suy theo từng feature nữa — đúng hành vi RimWorld designator
  (chọn 1 loại việc cụ thể trước, kéo qua feature khác loại thì bị bỏ qua, không tự nhận việc khác).
- `WildTaskbarController.cs` doc-comment cập nhật (Orders giờ data-driven như Structure, không còn
  placeholder). `NotImplementedYet` hiện KHÔNG còn nút nào dùng, giữ làm helper cho tương lai.
- Guide `Nizhangjie/M13_ZONE_TASKBAR_GUIDE.md` viết lại toàn bộ phần Orders (§0/§1.1/§1.5-§1.5d/§1.7/§2/§3) —
  thêm §1.5b hướng dẫn dựng prefab `SlotOrder` MỚI (chưa có sẵn, khác Structure tái dùng `ItemSlot`), §1.5c
  dựng `Panel_Orders` (Content grid + 1 nút "Huỷ" cố định, KHÔNG còn nút loại việc tay).
- **CHƯA làm** (để sau nếu cần): bulk-undesignate (kéo vùng để HUỶ hàng loạt, hiện chỉ add), Zone chuyển
  data-driven (chỉ Orders đợt này — user chốt), icon thật cho từng WorkGiverDef (đang glyph "📋" chung cho
  mọi loại, chờ user tự vẽ rồi gán `iconAddress` JSON).
- **CHƯA test tay trong Unity** — user cần tự dựng prefab `SlotOrder` + `Panel_Orders` theo guide mới (Zone/
  Structure panel wiring cũ KHÔNG bị ảnh hưởng, chỉ Orders đổi).

**BUG THẬT phát hiện + sửa 2026-07-12 (cùng phiên, user tự tìm ra qua test) — designation kẹt trên feature đã
cạn (Depleted):** `WorkAssigner.FindBestAssignment` (chỗ ghép pawn) đã lọc `feature.Depleted` từ trước, nhưng
3 CHỖ TẠO designation lại KHÔNG lọc — feature cạn vẫn NẰM LẠI trong `data.Features` (chờ hồi, đúng thiết kế
regrow) nên vẫn "quét thấy" được, dán designation lên → không pawn nào bao giờ nhận (bị Depleted-filter chặn)
→ icon "chờ việc" kẹt vĩnh viễn tới lúc feature hồi lại. 3 đường tới bug + đã sửa cả 3:
1. `CommitDesignateDrag` (tool quét khối MỚI, nguồn user báo bug) — thêm `if (feature.Depleted) continue;`.
2. `ToggleDesignation` (single-click context menu, "Designate") — thêm guard tương tự (phòng thủ thêm; UI
   `FeatureProvider.cs:53` vốn đã ẩn nút khi Depleted nên đường này khó xảy ra hơn, nhưng an toàn hơn không thừa).
3. **`WildGatherSystem.Complete`** (chỗ feature CHUYỂN sang Depleted/bị xoá) — thêm
   `data.Designations.Remove(f.Position)` — dọn designation TỒN ĐỌNG bất kể ai tạo ra nó. Case cụ thể: player
   đánh dấu 1 feature cho AI làm, RỒI tự tay gather trực tiếp feature đó TRƯỚC khi pawn kịp tới → feature cạn
   nhưng designation cũ (do path khác tạo, không phải AI path) không tự dọn nếu thiếu chỗ này. **LƯU Ý: sau bug
   #2 dưới đây, `Complete` giờ là điểm XOÁ THẬT DUY NHẤT cho cả đường AI** (không còn "AI tự Remove trước" như
   ghi ban đầu — xem bug #2).

**BUG THẬT #2 phát hiện + sửa 2026-07-12 (cùng phiên, user tự tìm ra) — nhiều pawn hội tụ vào 1 feature ở xa:**
`JobGiver_Designation.Execute` XOÁ designation NGAY lúc pawn nhận việc (trước khi tới nơi) — trong lúc pawn còn
đang đi, feature CHƯA Depleted (chưa harvest xong) nên bug #1's Depleted-guard không chặn được: player vẫn dán
được designation MỚI lên feature đó → pawn khác nhận → 2+ pawn cùng đi tới 1 feature. Hệ quả phụ nghiêm trọng
hơn: pawn ĐẾN SAU sẽ bị `WildGatherSystem.Complete` return sớm (`if (f.Depleted) return;`, feature đã bị pawn
đầu harvest xong) TRƯỚC khi gọi `onComplete()` → pawn đó kẹt "busy" vĩnh viễn trong `PawnAiBrain._active`,
không bao giờ nhận việc mới nữa.

**User tự đề xuất giải pháp đúng** (giữ mark lại, đánh dấu "đã có pawn lo" thay vì xoá — vừa chặn re-mark vừa
trực quan hơn "feature này đang được xử lý"). Đã code theo đúng hướng:
- `Designation.cs` +property `Claimed` (bool, mutable, mặc định false).
- `WorkAssigner.FindBestAssignment` +filter `if (designation.Claimed) continue;` — pawn rảnh khác không ghép
  được vào designation đã có pawn nhận.
- `JobGiver_Designation.Execute` đổi từ `Designations.Remove(...)` → `designation.Claimed = true` (giữ object
  reference từ `WorkAssignment.Designation`, mutate trực tiếp — KHÔNG cần GetAt lại). Xoá THẬT dời sang
  `WildGatherSystem.Complete` (đã có sẵn từ bug #1 fix, giờ là điểm xoá chính thức thay vì chỉ dọn-dư).
- `WildTilemapRenderer.PaintDesignations` +field `_claimedTint` (Color, mặc định xanh dương) — designation
  Claimed=true đổi màu icon, phân biệt trực quan "chờ pawn" (vàng/trắng) vs "đã có pawn lo" (xanh).
- Test mới trong `WorkAssignerTest.cs` (test 10-11): Claimed → null; mix Claimed+chưa-Claimed → vẫn ghép được
  cái chưa Claimed (không bỏ sót ứng viên hợp lệ khác).
- **Chưa làm** (ngoài phạm vi, không phải quên): Cancel API thật khi player Undesignate 1 ô ĐÃ Claimed — hiện
  vẫn xoá được designation dù pawn đang đi tới (pawn vẫn tiếp tục đi, không bị gọi lại) — hành vi cũ, không
  phải bug MỚI, cần Cancel API cho `WildGatherSystem`/`WildPlayerToken.MoveTo` (đã ghi nhận từ trước, xem §4
  M13_PAWN_AI_BRAIN_DESIGN.md) mới sửa triệt để được.

**CHƯA test tay lại sau CẢ 2 bug fix** — user cần re-test: (1) designate 1 feature → gather trực tiếp feature
đó KHÔNG chờ pawn → icon biến mất ngay không kẹt; (2) designate 1 feature Ở XA → trong lúc pawn đang đi, thử
dán designation lên CÙNG feature đó lần nữa → phải KHÔNG dán được (hoặc icon đổi màu xanh báo đã có pawn lo).

**BUG THẬT #3 phát hiện + sửa 2026-07-12 (cùng phiên, user tự tìm ra) — pawn "own nhiều job" cùng lúc:**
Nguyên nhân KHÁC bug #2 (không phải 2 AI-pawn cùng designation) — mà là **2 ĐƯỜNG GỌI ĐỘC LẬP cùng gọi
`WildGatherSystem.Begin` trên CÙNG 1 token**: (a) player right-click chọn verb Hái/Đào TRỰC TIẾP
(`WildSceneInteraction.HandleAction` → `_gather.Begin`, KHÔNG đụng `PawnAiBrain._active`) và (b) AI gán
designation qua `JobGiver_Designation.Execute` → `_gatherSystem.Begin`. Vì `PawnAiBrain` chỉ biết "bận" qua
`_active` (chỉ AI-path ghi vào đó), 1 pawn đang được player điều khiển tay VẪN bị AI coi là rảnh → gán thêm
designation khác → gọi `MoveTo` ĐÈ LÊN lệnh player đang chạy.

**Fix — dời "bận" xuống đúng lớp thực thi CHUNG (`WildGatherSystem`), không chỉ ở `PawnAiBrain`:**
- `WildGatherSystem.cs` +field `_busy: HashSet<WildPlayerToken>` + method public `IsBusy(token)`. `Begin()`
  guard đầu hàm: token đã `_busy` → log warning + return (không Move đè). `_busy.Add(mover)` ngay trước
  `MoveTo`; `Complete()` luôn `_busy.Remove(actor)` NGAY DÒNG ĐẦU (trước mọi early-return khác) — hết bận
  đúng lúc pawn TỚI NƠI, bất kể gather sau đó có thành công hay không.
- `PawnAiBrain.HandleTick` — nhánh pawn KHÔNG có trong `_active` (brain nghĩ là rảnh) giờ thêm check
  `if (_gatherSystem.IsBusy(token)) continue;` TRƯỚC khi `EvaluateTiers` — chặn AI gán job cho pawn đang
  bận vì lý do KHÁC (player lệnh tay).
- Đúng yêu cầu user: brain vẫn QUÉT LIÊN TỤC mỗi tick (không đổi), chỉ THÊM điều kiện "đang bận thì bỏ qua"
  — chờ job/lệnh hiện tại xong mới nhận job mới.
- **Chưa làm/ngoài phạm vi:** `_active` (AI ownership) và `_busy` (WildGatherSystem, physical-busy) là 2
  tracker TÁCH BIỆT có chủ đích (không hợp nhất) — `_active` phục vụ interrupt-theo-tier (Tier 2 tương lai),
  `_busy` phục vụ "vật lý đang di chuyển/khai thác" bất kể nguồn gọi. Không test Editor (MonoBehaviour, cần
  Play Mode — đúng convention dự án).

**CHƯA test tay** — user cần re-test: chọn 1 pawn → lệnh tay verb Hái/Đào 1 feature xa → TRONG LÚC đang đi,
mở Orders panel dán designation khác lên feature khác → pawn phải TIẾP TỤC đi tới feature đầu tiên, KHÔNG bị
đổi hướng; designation thứ 2 phải được 1 pawn KHÁC (nếu có) hoặc CHÍNH pawn này nhận SAU KHI xong việc đầu.

**BUG THẬT phát hiện + sửa 2026-07-24 (sau khi [[project_m19_pawn_idle]] có nội dung) — `PawnAiBrain.
ForceJob` (Store/Take/Open/PickUp/kéo-thả, xem [[project_dragdrop_colonist]]) từ chối gần như MỌI lệnh
với log "đang bận":** `ForceJob` cũ `return false` thẳng nếu `_active.ContainsKey(token)` — hợp lý lúc
Tier 4/5 CHƯA có nội dung (pawn rảnh thật thì `_active` luôn rỗng), nhưng từ khi M19 có nội dung, pawn
rảnh GẦN NHƯ LUÔN có 1 job idle (tu luyện/lang thang) chiếm `_active` → mọi lệnh trực tiếp (Tier 0, ý
định LUÔN thắng) bị chặn dù pawn chỉ đứng thiền/lang thang, không có gì thật cần bảo vệ. Triệu chứng
user báo: "Mine" (qua context menu) vẫn chạy được (đi qua `EvaluateTiers` interrupt bình thường, Tier 3
tự ngắt Tier 4/5), nhưng "Store"/kéo-thả (qua `ForceJob`, KHÔNG qua tier-scan) luôn bị từ chối.

**Fix:** `ForceJob` giờ Cancel job `_active` hiện tại (BẤT KỲ tier nào) rồi nhường chỗ, thay vì return
false — mirror ĐÚNG logic interrupt `HandleTick` đã dùng (`runningJob.Cancel?.Invoke(token); StartJob(...)`).
**Vẫn giữ nguyên gate `_gatherSystem.IsBusy`** (kiểm tra TRƯỚC, không đổi) — đường CŨ (player click tay
verb gather/`JobGiver_Designation`) không có Cancel API sạch (chỉ callback-khi-xong, `Cancel` field luôn
null), ép chồng sẽ tái hiện bug "pawn own nhiều job" 2026-07-12. An toàn vì: mọi job JobDriver-based
(Construction/Haul/BuildStructure/HaulStructure/StandingOrder/Wander) ĐỀU implement `Cancel` sạch; còn
`JobGiver_Designation` (Cancel=null) LUÔN đồng thời giữ `_gatherSystem._busy=true` suốt lúc đang chạy
(Begin gọi `_busy.Add` synchronous cùng lúc `StartJob` gọi `Execute`) nên gate `IsBusy` chặn được TRƯỚC
khi code chạm tới nhánh Cancel `_active` — không có cửa sổ nào Designation job "active" mà "not busy".

**CHƯA verify Play Mode sau fix.**

**BUG THẬT CRUCIAL #2 phát hiện + sửa 2026-07-25 (user tự test — cho pawn đi xuyên map, phát hiện nó tự
dừng giữa đường)** — verb "Move" (tách riêng 2026-07-24, xem CLAUDE.md M18) gọi THẲNG `WildPlayerToken.
MoveTo`, **KHÔNG hề đăng ký vào `_active`/ForceJob** (chủ đích lúc đó: "di chuyển thô, không cần hành động
sau khi tới nơi"). Hậu quả: brain KHÔNG BIẾT pawn đang đi — SlowTick kế (≤1s sau) thấy pawn "rảnh" (không
có trong `_active`, `_gatherSystem.IsBusy` cũng không track đường Move vì đây không phải gather) → tự gán
job idle (Tier 4/5, [[project_m19_pawn_idle]]) → `Toils.Goto` gọi `MoveTo` KHÁC đè lên path đang chạy →
pawn đi được vài bước rồi rẽ ngang làm "Cultivating"/"Wandering". Lỗ hổng có TỪ 2026-07-24, chỉ lộ rõ SAU
KHI M19 có nội dung thật — cùng họ với bug #1 ở trên.

**Fix:** `WildSceneInteraction.HandleMove()` (mới, thay code inline cũ trong case "Move") — giờ đi qua
`_brain.ForceJob` (Tier 0, mirror Store/Take/PickUp) thay vì gọi `MoveTo` trực tiếp — 1 toil `Toils.Goto`
duy nhất, không có hành động gì sau khi tới (giữ đúng hành vi cũ, chỉ thêm việc ĐĂNG KÝ vào `_active`).
Có fallback `mover.MoveTo(target)` thẳng nếu `_brain` chưa wire (tránh Move chết hẳn nếu scene thiếu field).

**Đã audit toàn bộ chỗ gọi `WildPlayerToken.MoveTo` trực tiếp khác trong dự án** — chỉ "Move" bị hở hoàn
toàn: `WildGatherSystem.Begin` (đường gather cũ) đã có `_busy` HashSet bảo vệ từ trước; `Toils.Goto` nội
bộ luôn chạy trong context `_active` đã đăng ký; `blocker.MoveTo` (đẩy pawn đứng lì khỏi công trường,
`JobGiver_Construction`/`BuildStructure`) VẪN CÒN hở tương tự nhưng quy mô nhỏ (đẩy đúng 1 ô, tần suất
thấp — CHƯA sửa, chấp nhận được, không phải thứ user báo).

**CHƯA verify Play Mode sau fix Move.**

Liên quan: [[project-pawn-system-masterplan]], [[project-m13-gameplay-design]], [[project_m19_pawn_idle]]
(nguyên nhân gốc — Tier 4/5 có nội dung mới lộ bug), [[project_dragdrop_colonist]] (nơi ForceJob được gọi).
