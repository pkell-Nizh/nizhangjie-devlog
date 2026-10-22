---
name: project-pawn-system-masterplan
description: M13 Pawn System masterplan (Designation/WorkGiver/Scheduler/Gather-regrow/Hunt/Building/Hauling) — doc + tiến độ code
metadata: 
  node_type: memory
  type: project
  originSessionId: 3b73d546-2d35-4735-8455-f1b5e04e0757
---

Doc plan đầy đủ: `Assets/Scripts/Nizhangjie/M13_PAWN_SYSTEM_MASTERPLAN.md` (viết 2026-07-04, tham khảo RimWorld/
Amazing Cultivation Simulator/Dungeon Settlers). 6 hệ con: Designation Layer, Gather mở rộng, Hunting, Building,
Hauling, PawnWorkScheduler (vòng lặp AI cốt lõi).

**ĐÃ CODE 2026-07-04 (2 phase đầu, user chọn làm trước, KHÔNG theo thứ tự §9 gốc):**
1. **Gather regrow + phân biệt Tree/Ore** — user chốt: CẢ cây LẪN quặng đều hồi lại theo thời gian (không chỉ
   cây). `MapFeatureCategory` +Tree(5)/+Ore(6) (append-only); `MapFeatureDefinition` +`harvestable`/+`verbOverride`/
   +`regrowDays`; `MapFeature.RegrowAtDay`; `FeatureProvider.GatherVerb()` data-driven (category→verb mặc định:
   Tree→Chặt/Ore→Đào/Resource→Hái, override khi lệch chuẩn vd Câu/Nhặt) thay switch cứng theo featureType cũ;
   `WildGatherSystem` thêm `GameClock.OnDayTick` quét regrow (feature Depleted giữ nguyên tại chỗ thay vì xoá,
   tự hồi khi tới hạn — ô chặn (Tree/Ore) LUÔN mở lối ngay khi khai thác, tự khoá lại khi regrow xong).
   Áp dụng regrowDays/category thật cho toàn bộ feature liên quan (DefaultSet + 7 JSON local_features).
2. **Hauling/Storage MVP thủ công** — user chốt lý do giữ túi cá nhân riêng: pawn đi viễn chinh/dungeon vùng
   khác cần mang đồ loot theo mình, không lẫn kho map đang đứng. `WildStorageInstance` (POCO, Cell+Container
   dùng LẠI `PlayerItemBag` làm container +DisplayName) sống trong `LocalMapData.Storages`. Flow thủ công (CHƯA
   phải auto-haul AI): right-click ô trống → action "Đặt Kho Tạm" (TerrainProvider) → tạo storage; right-click ô
   có kho → `WildStorageProvider` (Kind.Storage mới) → "Cất Đồ"/"Lấy Đồ" (toàn bộ, qua `WildStorageOps.TransferAll`).
   `WildSceneInteraction.HandleAction` giờ dispatch theo `TileInteractable.Kind` (trước chỉ có nhánh gather).

**CHƯA làm (user chốt tạm hoãn):**
- **Hunting** — user nói rõ "chưa làm hunt bây giờ". BeastSpawner/BeastBehaviour vẫn CHƯA nối combat.
- **Building thật** (blueprint/construction) — user NHẮC RÕ: Building của City (M11, `Building`/`BuildingDefinition`/
  `WorkAssignment` gắn `CityLayout`/Sect flow) ⟂ Building của Wild là **2 THỨ KHÁC NHAU**, đừng mặc định tái dùng
  thẳng class City cho Wild khi tới lúc code phần này — cần đánh giá riêng.
- **Designation Layer + WorkGiver + PawnWorkScheduler** (vòng lặp AI cốt lõi thật — đệ tử tự đi làm không cần
  player click) — vẫn ở dạng plan, chưa code gì. Đây là phần lớn nhất còn lại theo đúng tinh thần "Phase 3 ⭐".

**Quyết định chốt 2026-07-04 (trả lời §10 doc, ghi vào doc §10+§13):**
1. Regrow: giữ nguyên feature, chỉ đổi `Depleted` (khớp code đã làm).
2. Storage v1: **nhiều** storage building, chọn **gần nhất** (không cần zone filter).
3. Combat săn: v1 KHÔNG auto-join party (solo), cân nhắc lại ở version sau.
4. Building placement: check passable + không đè feature/building khác, đủ v1.
5. **Party system — QUAN TRỌNG: KHÔNG phải làm mới, đã tồn tại từ trước M13** (`Sim/Parties/Party.cs` +
   `PartyRegistry.cs`, wire sẵn `LocalMapManager.PartyRegistry`, có `Disciple.CurrentPartyId`/`IsInParty`/
   `JoinParty`/`LeaveParty`, có test `M13PartyTest.cs` 9 case từ trước). Ban đầu mình tưởng nhầm là chưa code và
   định tạo `ExpeditionParty` mới — SAI, đã sửa lại doc. `Party.Name` vốn đã player-settable, `WorldPos`
   (Vector2Int world-tile) vốn đã có nhưng là stub chưa dùng.

**ĐÃ CODE thêm 2026-07-04 (đợt 2, an toàn, không đụng flow đang chạy):**
`Party.IsAt(x,y)`, `PartyRegistry.GetPartyAt(x,y)`/`RenameParty(id,name)`/`SetWorldPos(id,pos)`,
`PartyRenamedEvent` mới, + test tương ứng trong `M13PartyTest.cs`.

**GAP THẬT còn treo — "Global Hub re: Party" (user chốt: DỪNG LẠI, KHÔNG code ngay, chỉ note):**
`DiscipleRegistry`/`PartyRegistry` hiện CHỈ sống trong `LocalMapManager` (scene City), `new` MỚI mỗi lần Awake
(không persistent). Scene Wild (`WildSceneBootstrap`/`WildPartyTestSpawner`) spawn pawn thẳng từ
`DiscipleDatabase`, KHÔNG hề tạo/tra `Party` nào. User muốn hệ "bến cảng/dock toàn cục" cho Party (nhiều đội hoạt
động song song, áp dụng nhiều nội dung) nhưng **tự nhận đây là quyết định rủi ro, chưa đủ thông tin để chốt ngay**
— đụng sâu Save/Load (`SaveManager`/`GameSaver`/`WorldMapSceneBootstrap`/`MainMenuController`, ~20 file). Đã ghi
chi tiết hướng khả dĩ vào doc §13.2b (KHÔNG phải milestone, chỉ note): có sẵn pattern bridge scene-to-scene
`WorldEntryContext` (static, `World/UI/WorldMapHUD.cs`) — hướng tự nhiên là mở rộng field đó thêm `PartyId`, KHÔNG
nhét `DiscipleRegistry`/`PartyRegistry` thẳng vào `ServiceLocator` (chính `ServiceLocator.cs` ghi chú "Disciple/
Building/CombatUnit dùng registry riêng, không cho vào đây"). Trước khi code thật: phải đọc kỹ
`SaveSystem/SaveManager.cs` + `UI/Sim/GameSaver.cs` để biết roster đệ tử load/save thế nào, registry có bị tạo
lại mỗi scene không. **Test các phần lẻ trước** (Party helper mới đã có test riêng), việc cầu nối để dành sau.

**ĐÃ CODE 2026-07-05 — Zone RimWorld-style (Vùng Trồng + Vùng Câu, KHÔNG stockpile) + Taskbar tool:**
User chốt: chỉ Growing zone + Fishing zone (kho là building thật sau này, không cần stockpile zone). Taskbar =
UX thật trong game (không phải debug), giống thanh dưới RimWorld.

Phát hiện quan trọng: `LocalMap/Zoning/Zone.cs`/`ZoneType.cs` đã có SẴN từ trước (ghi chú cũ "ZoneManager (M4+)
sẽ manage") nhưng CHƯA AI IMPLEMENT — hoàn toàn chết, không wire vào `LocalMapManager`. Đã viết
`ZoneManager.cs` (mới, implement đúng TODO cũ) + append `ZoneType.WildGrowing=10`/`WildFishing=11` — TÁI DÙNG
hạ tầng cũ, không tạo hệ Zone song song.

Đã code: `MapFeatureCategory.Crop=7` (verb "Thu Hoạch"); `LocalMapData.Zones` (ZoneManager);
`World/Wild/Zones/WildZoneOps.cs` (`PlantGrowingZone` TÁI DÙNG 100% hệ regrow có sẵn — trồng = tạo MapFeature
Depleted=true + hẹn RegrowAtDay, KHÔNG tick mới; `CatchFish` tức thời không cần di chuyển);
`WildFishZoneProvider` (ô nước trong Vùng Câu → "Câu"); `InteractableKind.Zone=7`; `WildSceneInteraction`
+`WildToolMode` (drag-rect vẽ zone, huỷ bằng right-click/Esc) +3 wrapper method
(`SelectMoveTool`/`SelectGrowingZoneTool`/`SelectFishingZoneTool`) để nút taskbar dán qua **OnClick Inspector**
(theo đúng [[feedback-ui-button-wiring]] — KHÔNG AddListener trong code); `WildTilemapRenderer` +`HighlightRect`/
`PaintZones`. Test: `Tests/Editor/M13ZoneTest.cs` (6 case). Guide dựng UI:
`Assets/Scripts/Nizhangjie/M13_ZONE_TASKBAR_GUIDE.md` (chỉ cần 3 Button, KHÔNG cần script controller mới).

CHƯA làm: đệ tử tự làm trong zone (cần PawnWorkScheduler), palette chọn loại crop khác, giới hạn đất trồng theo
tile type, giới hạn số cá Vùng Câu.

**Cập nhật 2026-07-05 (cùng phiên, SỬA LẠI LẦN 2) — Taskbar đúng 3 TẦNG kiểu RimWorld:**
Bản nháp đầu (2 tầng: nút "Zone"/"Building" ngang hàng) SAI — user sửa lại: **1 nút gốc "Building"** (luôn hiện
trên taskbar) → bấm mở **cột danh mục** (Zone/Orders/Structure) → bấm 1 danh mục → hiện **panel icon** riêng
(chỉ 1 panel tại 1 thời điểm, cột danh mục vẫn hiện khi đổi qua lại danh mục).

`WildTaskbarController` (`World/Wild/UI/`, viết lại) — `ToggleBuildingPanel()` (nút gốc) +
`ShowZonePanel()`/`ShowOrdersPanel()`/`ShowStructurePanel()`. Chỉ quản lý SetActive, KHÔNG chứa logic nút:
- **Zone** → logic THẬT (Vùng Trồng/Vùng Câu/Huỷ qua `WildSceneInteraction`, đã code từ trước).
- **Orders** → placeholder — đánh dấu hành động lên feature (Chặt/Đào/Săn...) CHỜ đệ tử đến làm sau. Cần
  Designation Layer (§2 masterplan, CHƯA code).
- **Structure** → placeholder — xây công trình có công dụng thật (Lều Trại/Kho...). Cần Building blueprint
  (§6 masterplan, CHƯA code).

Cả Orders/Structure chỉ gọi `NotImplementedYet(string what)` → log console, không làm gì thật. Guide
`M13_ZONE_TASKBAR_GUIDE.md` §1 đã viết lại toàn bộ theo cấu trúc 3 tầng này (bảng wire OnClick đầy đủ).

**Cập nhật 2026-07-05 (cùng phiên, lần 3) — Panel Structure đổi thành DATA-DRIVEN** (user chỉ rõ: Zone/Orders
cố định vì "tiện nhanh + cốt lõi không đổi theo bối cảnh", nhưng Structure cần "bao nhiêu công trình cũng được"
→ phải đọc JSON, không hard-code nút). Đã tạo:
- `Data/Definitions/WildBuildingDefinition.cs` (mới, JSON-first, KHÔNG kế thừa `LocalMap/Buildings/
  BuildingDefinition.cs` của City — đúng cảnh báo §6.1 "2 thứ khác nhau"). 3 JSON ví dụ trong
  `Data/wild_buildings/` (+ mirror StreamingAssets): `building_camp_tent`/`building_storage_shed`/
  `building_sect_hall`.
- `World/Wild/UI/WildStructurePanel.cs` (mới) — TÁI DÙNG `InventorySlotUI`/`ItemSlotView` có sẵn (prefab
  `Assets/Prefabs/UI/CharacterMenu/Inventory/ItemSlot .prefab`, KHÔNG tạo prefab riêng), pattern y hệt
  `MaterialsPanel.cs` (grid + detail panel). Đây là panel DUY NHẤT dùng Instantiate/code-driven trong cả bộ
  taskbar — đúng lý do kỹ thuật: list động (số building thay đổi theo JSON/mod) thì AddListener/code thắng
  OnClick Inspector, còn Zone/Orders cố định thì Inspector vẫn đúng chỗ hơn.
- Nút "Xây" trong detail panel vẫn chỉ `PlaceSelectedNotImplemented()` → log, KHÔNG đặt building thật (Building
  blueprint §6 vẫn chưa code, đúng scope ban đầu).

**QUAN TRỌNG — xem [[project-global-structure-hub]] trước khi làm Building blueprint §6 thật:** user đề xuất
building = `functionId` (trỏ core system dùng chung — AlchemySystem/ForgeSystem/housing...) + capacity/efficiency
đọc JSON, thay vì viết logic riêng từng loại building. Có bằng chứng ủng hộ (`BuildingEfficiencyTable.cs` City
đang hardcode `.Contains("linh_dien")` — đúng nợ kỹ thuật cần sửa). CHƯA code hub thật — chỉ mới đề xuất field
data (`functionId`/`capacity`/`efficiencyMultiplier`) thêm vào `WildBuildingDefinition`, chờ user chốt.

**ĐÃ CODE 2026-07-11 — Capability layer (tiền đề bắt buộc trước Designation/WorkGiver §2/§9):**
User chốt: trước khi làm "applied job lên feature" (RimWorld-style), cần cơ chế Capability trả lời "đệ tử này
LÀM ĐƯỢC job X không" (gate) + "giỏi cỡ nào" (level, mang chỉ số) — tham khảo RimWorld WorkType/SkillDef +
Amazing Cultivation Simulator. Đã code:
- `Data/Definitions/CapabilityDef.cs` (mới, kế thừa `Def` — moddable, JSON `Data/capabilities/` + mirror
  StreamingAssets). Field: `category` (free, chỉ UI), `primaryStat`/`secondaryStat` (tên StatType, TÁI DÙNG
  đúng pattern `BuildingEfficiencyTable.primaryStat`), `requiresTrait` (rỗng = ai cũng có sẵn; set = cần trait
  đó mới unlock — vd craft chuyên sâu sau này), `maxLevel`/`xpPerSuccess`/`xpPerFailure`/`xpPerLevelStep`
  (leveling knob, mirror `CraftSkill` 1:1 — XP cần lên level = Level × xpPerLevelStep).
- `Sim/Disciples/Capabilities/CapabilitySet.cs` (mới) — SỞ HỮU BỞI `Disciple` trực tiếp (như Traits/Skills,
  KHÔNG phải registry rời kiểu `CraftSkillRegistry`, vì Disciple đã tự quản Traits/Skills/ItemBag). API:
  `HasCapability(id, owner)` (gate), `GetLevel`/`GetXP`, `GetEfficiency(id, stats)` (statEff từ core stat ×
  levelMult học được, 0.1/level giống CraftSkill), `RecordWork(id, success)` (cộng XP, tự lên level, trả true
  nếu lên level).
- `Disciple.cs` +field `Capabilities` + 3 method tiện `HasCapability`/`GetCapabilityEfficiency`/
  `RecordCapabilityWork` (delegate xuống CapabilitySet, tự truyền `this`/`Stats`).
- `GameDatabase.LoadAll()` +`LoadType<CapabilityDef>("capabilities")`.
- 4 capability khởi điểm (JSON, id KHÔNG prefix — loader tự stamp `core:`): `mining` (Foundation, theo đúng doc
  XML `StatType.cs` "Khai khoáng"), `harvest` (Comprehension, "Hái thuốc"), `grow_plant` (Foundation+Comprehension),
  `craft` (Force+Comprehension, giống mapping `re_khi_cu` legacy trong BuildingEfficiencyTable). Cả 4 đều
  `requiresTrait=""` — mọi đệ tử làm được ngay, chỉ khác hiệu suất.
- Test: `Tests/Editor/CapabilityTest.cs` (8 case, menu `Nizhangjie/Run Capability Tests`) — gate, level/XP init,
  RecordWork tăng XP/lên level, GetEfficiency statEff×levelMult, cap ở maxLevel.

**CHƯA làm (đúng scope phiên này — chỉ cơ chế, chưa nối job thật):** WorkGiver/Designation chưa đọc
`requiredCapabilityId` từ job def nào — đó là bước TIẾP THEO khi làm "applied job lên feature" thật (Designation
Layer §2 masterplan). `WildGatherSystem` hiện tại (player tự click) CHƯA gọi `RecordCapabilityWork`/
`GetCapabilityEfficiency` — cần wire khi chuyển sang đệ tử tự làm qua AI.

**ĐÃ CODE 2026-07-11 (cùng phiên, lần 2) — Designation Layer + WorkGiver (Phase 3 ⭐ masterplan, MVP cho
Mining+Harvest, tham khảo RimWorld WorkTypeDef/WorkGiverDef/JobGiver):**
Kiến trúc 3 lớp tách rõ (giống RimWorld): **Designation** (ý định player, không biết ai làm) ⟂ **WorkGiverDef**
(JSON, "công thức việc": capability nào + feature category nào) ⟂ **WorkAssigner** (logic ghép, plain C# test
được) + **PawnWorkScheduler** (MonoBehaviour orchestrator MỎNG, tick 1/s qua GameClock.OnSlowTick).
- `World/Wild/Designations/Designation.cs` + `DesignationManager.cs` (mới) — CRUD theo Vector2Int, 1 ô = 1
  designation. Gắn vào `LocalMapData.Designations` (field mới, mirror `Zones`/`Storages`).
- `Data/Definitions/WorkGiverDef.cs` (mới, kế thừa `Def`) — `requiredCapabilityId` (resolve qua
  `IDefResolver` như `BeastSpeciesDefinition`, fail-fast nếu CapabilityDef không tồn tại) + `targetCategories`
  (List&lt;string&gt; tên `MapFeatureCategory`, validate parse được trong `ConfigErrors`) + `MatchesCategory()`.
  2 JSON khởi điểm (`Data/work_givers/` + mirror StreamingAssets): `mine_ore` (Ore → core:mining),
  `harvest_wild` (Tree/Resource/Crop → core:harvest). **GrowPlant + Craft CHƯA có WorkGiver** — GrowPlant cần
  scan ô TRỐNG trong zone (không phải feature có sẵn, chiến lược scan khác hẳn), Craft đã có
  `CraftingQueue`/`CraftSkillRegistry` riêng ở City — để dành đợt sau, không ép vào layer này.
- `World/Wild/Work/WorkAssignment.cs` (struct) + `WorkAssigner.cs` (mới, plain C# static, KHÔNG đụng
  MonoBehaviour → test thẳng bằng data giả) — `FindBestAssignment(pawn, pawnCell, designations, featureAt)`:
  lọc `pawn.HasCapability` + feature chưa Depleted + `WorkGiverDef.MatchesCategory`, chọn GẦN NHẤT (Manhattan
  distance, đủ dùng MVP — không cần A* thật chỉ để so khoảng cách).
- `World/Wild/Work/PawnWorkScheduler.cs` (mới, MonoBehaviour orchestrator) — mỗi SlowTick (1s) quét
  `WildPlayerToken` rảnh (chưa `_busy` + `Disciple.IsAvailableForWork`) qua `FindObjectsByType` (chưa có
  registry pawn tập trung trong Wild scene, xem gap ở trên — chấp nhận được vì chỉ vài pawn/scene, giống
  `WildPartyTestSpawner` đã làm tương tự), ghép qua `WorkAssigner`, rồi **TÁI DÙNG NGUYÊN**
  `WildGatherSystem.Begin` để đi + khai thác (KHÔNG viết lại di chuyển/gather — chỉ thêm optional
  `Action&lt;MapFeature,WildPlayerToken&gt; onComplete` vào `Begin`/`Complete` để scheduler biết lúc xong → giải
  phóng pawn + gọi `RecordCapabilityWork`).
- Test: `Tests/Editor/WorkAssignerTest.cs` (9 case, menu `Nizhangjie/Run WorkAssigner Tests`) — CRUD
  Designation, MatchesCategory, gate bởi trait, category mismatch, Depleted, match hợp lệ, chọn gần nhất.

**CHƯA làm (rõ ràng, không phải quên):** UI để player TẠO designation thật (Orders panel trong
`WildTaskbarController` vẫn còn `NotImplementedYet` — cần tool giống Zone drag-rect nhưng click-to-mark từng
ô/feature) — layer BÊN DƯỚI (Designation/WorkGiver/Scheduler) đã chạy được và có test, chỉ thiếu lối vào từ
UI. Không có registry pawn tập trung Wild scene (dùng FindObjectsByType tạm — nếu số pawn tăng nhiều mới cần
tối ưu). Không có persist designation qua save/load. GrowPlant/Craft WorkGiver để đợt sau (lý do nêu trên).

**ĐÃ CODE 2026-07-11 (cùng phiên, lần 3) — UI tạo designation qua context menu CÓ SẴN (KHÔNG cần Orders
panel/taskbar riêng — dùng đúng flow right-click→InfoPanel→nút Action đã có cho gather):**
- `FeatureProvider.cs` +action "Designate"/"Undesignate" (verb tiếng Anh, đúng
  [[feedback-verb-strings-english]]) — chỉ hiện nếu `WorkGiverDef.FindForCategory(f.Category)` tìm được def
  khớp (mining/harvest hiện có; feature loại khác — Beast/Hazard/Decoration/PoiEntrance — KHÔNG hiện nút vì
  chưa có WorkGiver nào nhận). Nút đổi nhãn theo `data.Designations.HasAt(f.Position)` (đã tag → "Undesignate").
- `WorkGiverDef.cs` +`FindForCategory(category)` static — quét `DataRegistry<WorkGiverDef>.GetAll()`, trả def
  ĐẦU TIÊN khớp category (đủ dùng vì hiện mỗi category chỉ 1 WorkGiver; nếu sau có nhiều WorkGiver/category
  cần UI chọn thật, method này phải đổi thành trả list).
- `WildSceneInteraction.cs` — `HandleAction` case Feature tách riêng verb "Designate"/"Undesignate" (pattern
  `when`) khỏi verb gather thường (Chop/Mine/...) → `ToggleDesignation()` mới, gọi thẳng
  `data.Designations.Add/Remove` + `WorkGiverDef.FindForCategory` rồi `_renderer.Render(data)` để icon hiện
  ngay (KHÔNG đợi tick).
- **Icon overlay** — `WildTilemapRenderer.cs` +layer Tilemap RIÊNG `_designations` (field mới, optional —
  Order phải NẰM GIỮA Features và Highlight, user tự tạo GameObject trong scene + wire Inspector, xem hướng
  dẫn dưới) + field `_designationIcon` (Sprite, vd user tự vẽ icon 64×64 ghi "[working]"). `PaintDesignations()`
  vẽ TRÊN layer riêng (không ghi đè sprite feature bên dưới) — CHƯA gán `_designationIcon` → fallback ô vàng
  bán trong suốt (giống mọi fallback "chưa có art" khác trong renderer này, y hệt pattern feature/ground).
  Icon **biến mất NGAY LÚC pawn nhận việc** (`PawnWorkScheduler.AssignWork` gọi `_renderer.Render(data)` sau
  khi remove designation) — KHÔNG đợi làm xong. Lý do chọn mốc này: đơn giản hoá (không cần track "đang làm"
  riêng khỏi "đã bị consume" — 1 nguồn sự thật `DesignationManager`), tránh bug 2 pawn cùng nhận 1 designation.
- `WildGatherSystem.Begin/Complete` +param `Action<MapFeature,WildPlayerToken> onComplete = null` (đã có từ
  lần trước, verb param truyền vào KHÔNG được dùng trong logic — thuần hiển thị/tương lai).
- `WildPartyTestSpawner.Awake()` +`DataRegistry<CapabilityDef>.LoadFromFolder("capabilities")` +
  `DataRegistry<WorkGiverDef>.LoadFromFolder("work_givers")` — **THIẾU BƯỚC NÀY THÌ NÚT DESIGNATE KHÔNG BAO
  GIỜ HIỆN** (`FindForCategory` quét registry rỗng → null) — dễ quên vì Wild scene không gọi
  `GameDatabase.LoadAll()` (không có LocalMapManager), tự load lẻ từng loại content cần.

**Setup Unity Editor CẦN LÀM TAY (chưa/không thể code hộ):**
1. Trong Hierarchy chỗ có `Tilemap_Ground/Tilemap_Features/Tilemap_Highlight` — tạo thêm 1 Tilemap con tên
   `Tilemap_Designations`, `TilemapRenderer.sortingOrder = 2` (Features=1, đẩy Highlight cũ lên 3 nếu đang =2).
2. Kéo `Tilemap_Designations` vào field `_designations` (Inspector) của component `WildTilemapRenderer`.
3. (Tuỳ chọn, có thể để trống trước — vẫn chạy được, chỉ fallback ô màu vàng) Tạo sprite 64×64 ghi
   "[working]" (Import Unity: Texture Type = Sprite (2D and UI)) → kéo vào field `_designationIcon`.
4. Tạo 1 GameObject mới, add component `PawnWorkScheduler` (mới, `World/Wild/Work/`), wire 3 field Inspector:
   `_bootstrap` (WildSceneBootstrap có sẵn trong scene), `_gatherSystem` (WildGatherSystem có sẵn),
   `_renderer` (WildTilemapRenderer có sẵn — optional nhưng nên gán để icon biến mất đúng lúc).
5. Test luồng: Play → right-click 1 ô có Linh Khoáng/Linh Mộc/Bụi Quả... → context menu → chọn feature → info
   panel hiện nút "Mine"/"Chop"/... VÀ nút "Designate" mới → bấm Designate → icon hiện (hoặc ô vàng nếu chưa
   gán sprite) → chờ ~1s (SlowTick) → pawn rảnh gần nhất tự đi tới + khai thác, icon biến mất ngay lúc pawn
   xuất phát.

**Quyết định chốt 2026-07-11 (cùng phiên, lần 4) — CellIndices cho WildMap: SAU, không phải bây giờ:**
User hỏi có nên áp CellIndices (`Cell[]` struct phẳng, xem [[project-cellindices]]) cho `LocalMap/Grid/
GridSystem.cs` (hiện `Tile[,]` — mảng 2D ĐẶC nhưng mỗi ô là 1 object heap riêng, khác World đã chuyển sang
struct AoS). Phân tích: CellIndices có lợi RÕ cho thuật toán quét gần hết map cùng lúc (flood-fill/region
detection kiểu RimWorld `RegionGrid`, temperature, fire spread — cache-locality thắng hẳn dereference con trỏ
rải rác) nhưng KHÔNG có lợi cho A* pathfinding hiện tại (bounded, không quét toàn map) hay
Designation/WorkGiver vừa code (quét list, không đụng grid). **Hiện M13 roadmap CHƯA có hệ nào quét toàn
map kiểu đó** → chưa cần refactor ngay.

User chốt 2 điều:
1. **Thời điểm**: test tay `PawnWorkScheduler`/Designation UI (setup Editor đã ghi ở trên) CHẠY ĐÚNG trước
   đã — KHÔNG refactor GridSystem cùng lúc (tránh vừa debug feature mới vừa debug nền tảng đổi).
2. **Quy mô khi làm**: FULL như World đã làm (`Cell[]` struct phẳng, sentinel zero-init, đổi MỌI call site
   — `LocalPathfinder`, mọi Wild provider, `WildTilemapRenderer`, `ZoneManager`, `WorkAssigner`...) — KHÔNG
   làm bản tối thiểu giữ API cũ.

**Việc CẦN LÀM TIẾP theo đúng thứ tự đã chốt:**
1. User tự test Designation/WorkGiver trong Unity Editor (theo setup 5 bước đã ghi ở mục lần 3 trên) — Claude
   không có quyền truy cập Unity Editor trong phiên này để tự chạy Play mode.
2. Sau khi xác nhận chạy đúng → quay lại làm CellIndices full-scope cho WildMap, viết plan riêng trước khi
   code (mirror `Nizhangjie/nizhangjie-cellindices-plan.md` đã có cho World) vì khối lượng đụng chạm lớn.

Liên quan: [[project-m13-gameplay-design]], [[project-cellindices]], [[project-global-structure-hub]].
