---
name: project-m13-gameplay-design
description: "Thiết kế M13 — gameplay+UI LocalMap thật (hybrid colony-sim tu tiên): quyết định chốt + 4 phase"
metadata: 
  node_type: memory
  type: project
  originSessionId: f2f6af37-1ebf-4190-aa56-c64b81240d01
---

**M13 — gameplay+UI LocalMap "thật" (ĐÃ CHỐT THIẾT KẾ, chưa code).** Doc đầy đủ:
`Assets/Scripts/Nizhangjie/M13_WILD_GAMEPLAY_DESIGN.md`. Khảo 3 game: RimWorld (designation+priority),
FF7R/FF15 (combat RTwP+ATB command), game tu tiên Tiancang (tooltip Qi/Tier/Quality, verb Harvest/Hunt/Explore).

**6 quyết định chốt với user:**
1. **Điều khiển HYBRID**: player avatar điều khiển trực tiếp (đi/đánh/solo) + đệ tử tự hành qua designation;
   trong combat ra lệnh tay kiểu FF7R (click đệ tử→move / click quái→đánh / kéo gauge ATB→xếp skill queue, pause/slow-mo).
2. **Combat LIỀN MẠCH**: localmap grid = combat grid HỢP NHẤT (yêu thú/sơn tặc đứng ngay trên map). Tái dùng
   `DamageCalculator`/hit shapes/ATB của `Combat/`, KHÔNG spawn CombatGrid/arena riêng. Beast feature→enemy unit.
   (Fallback: trận lớn có thể transition; lẻ thì seamless.)
3. **Wild map PHÂN TẦNG**: viễn chinh (mặc định) + xây TẠM (miễn ngoài `WorldTile.OwnerSettlementId` người khác)
   + **"cắm cờ khai địa"** (action riêng, điều kiện đất bằng+linh khí+ngoài lãnh địa) → tile thăng cấp thành BASE
   vĩnh viễn + bật bộ tool base. → 2 bộ tool (viễn chinh/xây-tạm luôn có ⟂ base đầy đủ chỉ sau cắm cờ).
4. **Tông môn = MAP VẬT LÝ (tile đã cắm cờ) ⟂ `Sim/Sect` (sổ cái offline)** — quan hệ STRUCTURE⟂STATE như City.
5. **Bí cảnh/dungeon = LocalMapData *authored*** (loader map dựng tay) — cùng đầu ra `LocalMapData` → render/inspect/
   combat/designation dùng lại 100%. Không hệ thống riêng.
6. **Tick: 10 TPS base + TIERING (TickFast mỗi tick combat/pawn-đang-đi; TickRare ~25 tick nhu cầu/AI; TickLong
   ~250 tick growth) + nội suy visual mỗi frame.** KHÔNG copy 60 TPS toàn cục (RimWorld 60 nhưng cứu bằng tiering).
   Deterministic fixed-timestep; tick/giây × GameClock speed (1×=10/2×=20/3×=30/pause=0). Combat chậm→nâng RIÊNG combat
   20–30 sub-tick. Wire vào tick sequence `LocalMapManager`.

**Designation/work**: verb Hái/Chặt/Đào/Săn/Khám phá/Dọn/Phá/Xây/Khuân/Tu luyện. Nối `MapFeature`(M12):
Resource→Hái/Đào, Beast→Săn, PoiEntrance→Khám phá; `TileInteractable.Actions`=nguồn nút→tạo designation. Đệ tử tự
nhận job gần nhất khớp JobSkill (Labor/Alchemy…), pathfind, yield. Priority đơn giản trước, lưới RimWorld sau. Data-driven.

**PHASE 2 ĐÃ CODE (hái/đào player-direct + kho thật, chưa verify compile — Editor mở):**
- **Kho dùng chung:** `Sim/Items/PlayerItemBag.cs` namespace **`Nizhangjie.Sim.Items`** (Dictionary id→count,
  `PlayerItemBag.Get()` get-or-create qua ServiceLocator → CÙNG kho mọi scene city+wild). Disk-save = follow-up.
  ⚠️ BẪY đã gặp+sửa: ĐỪNG đặt namespace `Nizhangjie.Sim.Inventory` — sẽ che type `Equipment.Inventory` khi code
  trong `Nizhangjie.Sim.*` (vd Disciple.cs) dùng `Inventory` không-qualify (CS0118). Dùng `Sim.Items`.
- **Material thật:** `Data/materials/{wild_berry,herb_blood_grass,ore_iron}.json` (+ spirit_herb có sẵn), có iconAddress=
  `Features/<id>` (dùng chung sprite feature). `MapFeatureDefinition` thêm `yieldItemId/yieldMin/yieldMax` (rỗng→=feature id).
- **Gather:** `World/Wild/Gather/WildGatherSystem.cs` — Begin(feature,verb) → token đi tới (ô feature hoặc ô kề nếu chặn)
  → Complete: +material vào bag + cạn feature (có depletedSpriteAddress→`MapFeature.Depleted=true` đổi sprite; else xoá
  khỏi data.Features + ô BlocksTile→Stone) → renderer.Render lại. `WildPlayerToken.MoveTo(target, Action onArrive)` mới.
  `FeatureProvider` điền `TileInteractable.Actions` (ore→Đào, resource khác→Hái, vein node skip). `WildInfoPanel` thêm
  nút action (ActionContent+prefab) + `SetActionHandler`. `WildSceneInteraction` wire `_gather`.
- **Hiển thị kho:** trạm HUD wild `World/Wild/UI/WildBagReadout.cs` + tab Vật Phẩm `UI/Sim/Inventory/MaterialsPanel.cs`
  + `MaterialSlotUI.cs` (gắn làm page trong TabBar CharacterMenu, OnEnable đọc bag — KHÔNG sửa CharacterMenuController).
- **CharacterMenu dùng chung:** user biến thành prefab, đặt vào wild scene + nút mở → Toggle(); cùng kho qua ServiceLocator.
- Guide wiring: `Nizhangjie/M13_PHASE2_GATHER_GUIDE.md`. NGOÀI phạm vi: đệ tử-AI (Phase 3), disk-save, mọc lại, đào bare Cliff.

**MÀN ITEM kiểu reference (tu tiên) — ĐÃ CODE + TÁI KIẾN TRÚC NHẤT QUÁN:** 1 tab Item → hàng chọn LOẠI
(Equipment/Manual/Consumables/Material/Map/Other) → dải chip filter HIỆN RÕ (user bỏ ý nút-ẩn-hover) → lưới + detail.
Files `UI/Sim/Inventory/`:
- **Contract chung `IItemView`** (`Bind(ItemViewContext)` + `GetFilters()` + `ApplyFilter()`) — MỌI sub-view implement
  để `ItemScreenController` điều phối ĐỒNG NHẤT (push). `ItemViewContext` struct {Disciple, EquipmentManager, EquipBag
  (IList<ItemInstance>), PlayerItemBag ItemBag, Action OnEquipmentChanged}. (Đã XOÁ `IItemSubView`.)
- `ItemScreenController`: type-row (mảng {id,Button,view}) + giữ `_context`; `Bind(ctx)` (CharacterMenuController gọi)
  + `Select(i)` bật view + `view.GetComponent<IItemView>().Bind(_context)` + FilterBar.Build (tự ẩn nếu rỗng); quick-settings stub.
- `ItemFilterBar` (chip động). `MaterialsPanel` implements IItemView (Bind lấy ctx.ItemBag, filter MaterialDefinition.category
  nhãn VN). `InventoryPanel` = **adapter mỏng** IItemView (Bind→Show cũ, GetFilters rỗng) — KHÔNG đụng logic equip/stat.
- **Manual = CÔNG PHÁP (功法) KHÔNG phải recipe** → `CongPhapPanel` list `DataRegistry<SkillDefinition>` (catalog),
  filter skillType. (XOÁ `RecipePanel` — sai khái niệm.) Consumables/Map/Other = stub.
- `CharacterMenuController`: thêm field `_itemScreen`; ShowCharacter dựng `ItemViewContext` → `_itemScreen.Bind(ctx)`
  (FALLBACK gọi `InventoryPanel.Show` cũ nếu `_itemScreen` trống). Equip slots + info + stats = CHROME (bind như cũ).
Guide: `Nizhangjie/M_ITEM_SCREEN_GUIDE.md`. ⚠️ Bẫy: PlayerItemBag namespace `Sim.Items` (KHÔNG `Sim.Inventory`).

**4 PHASE (mỗi phase chơi được):** ① HUD shell + inspect giàu (clock/tốc độ + cursor readout Lingqi/Soil + info panel
thêm Tier/Quality/Scenery/Qi) → ② Designation tools (palette + overlay, chưa AI) → ③ **Pawn work AI** (loop cốt lõi,
tick tiering ra đời) → ④ Combat seamless + hookup yield→kho/craft + cắm cờ→Sect bridge.

**PHASE 1 ĐÃ CODE (chưa verify compile — Editor mở):** (1) `WildTilemapRenderer` nâng **sprite-ready** — mảng
`GroundTiles[]` (TileType→Tile asset) + feature `spriteAddress` (Resources.Load, runtime Tile cache), fallback ô màu;
(2) `World/UI/LocalMapTimeBar.cs` (shared, đọc GameClock+GameCalendar, nhãn lịch+mùa-suy-từ-tháng + nút Pause/1x/2x/3x,
KHÔNG dính City/ví); (3) `World/Wild/UI/WildCursorReadout.cs` (hover ô→nền/passable/linh khí + biome/nhiệt/ẩm vùng,
cần `WildSceneBootstrap.SourceTile` mới thêm); (4) inspect giàu — thêm field `quality/scenery/qiGather` vào
`MapFeatureDefinition` + `FeatureProvider` emit dòng. Guide: `Nizhangjie/M13_PHASE1_HUD_GUIDE.md` (sprite/tile prep +
wiring table) + `Nizhangjie/M13_HUD_BUILD_GUIDE.md` (dựng HUD từng bước cầm-tay: Canvas→TimeBar→Title→CursorReadout
→InfoPanel→ContextMenu→nút, bảng wiring, troubleshoot). User TỰ chuẩn bị sprite + wire Editor.

**Renderer biến thể (đã thêm):** `WildTilemapRenderer.GroundTileEntry` đổi `tile`→`tiles[]` (mảng); >1 biến thể →
`PickGroundTile` random DETERMINISTIC theo ô (`FastNoise.Hash(localSeed^type, x, y)`). Gán 3 sprite cỏ vào 1 entry
Grass → đồng cỏ đỡ lặp, khỏi cần Random Tile của Unity.

**Lưu ý art TileType:** `Stone`=SÀN đá lát (passable), KHÔNG phải tường. Tường đá → gán `Cliff`/`Mountain`
(impassable) hoặc để dành làm sprite tường-building. Tường trong game = BUILDING (Occupant), không phải terrain tile.

**Bẫy compile đã gặp+sửa:** `WildTilemapRenderer` import cả `LocalMap.Grid` (Tile data) lẫn `UnityEngine.Tilemaps`
(Tile tilemap) → `Tile` ambiguous (CS0104). Phải qualify `UnityEngine.Tilemaps.Tile` khi CreateInstance.

**Art quyết định:** square top-down (KHÔNG isometric — code grid đã square, 3 ref đều square top-down). Tile/sprite
**64×64, PPU=64, Filter=Point, Compression=None**; ground tile gen ở góc **90° top-down**; vật/feature để dành góc
nghiêng ~56°. Vật to = bội số 64 (nhưng feature hiện render = 1 ô tilemap → tạm vẽ vừa 1 ô).

**MỞ RỘNG CHARACTER BASE + PARTY — ĐÃ CODE + COMPILE OK (2026-06-27):** 5 quyết định chốt với user:
(1) team **≤5 đệ tử thẳng**, linh thú "ăn ké" slot sau (Party.MaxSize=5). (2) **KHÔNG** làm Needs/Mood/Health
(không trauma-sim); HP/injury vẫn ở CombatStats. (3) Inventory cá nhân + kho tông môn "ném hết" (chưa code phần này).
(4) Đệ tử vào party → NGỪNG job tông môn, chờ lệnh. (5) Skill = `LearnedSkill {id, mastery, proficiency}` (KHÔNG List<string>).
**KEY: StatModifier + recompute-from-base ĐÃ CÓ SẴN** (`Stats/CharacterStats.cs` + `StatModifier.cs`, công thức
`(base+Σflat)×(1+ΣpctAdd)×Π(1+pctMul)`, `AddModifier`/`RemoveModifiersFromSource(source)`) — skill/buff CHỈ nối vào, KHÔNG xây mới.
Files mới: `Sim/Disciples/Skills/LearnedSkill.cs` (mirror Trait: Apply/Remove source `"skill:{id}"`, mastery 0-3
Sơ/Tiểu/Đại Thành/Viên Mãn, MasteryScale=1+0.15×lvl khuếch đại passiveModifiers, GainProficiency tự re-apply;
active đọc MasteryDamageScale), `Sim/Parties/Party.cs` (POCO + PartyState enum Idle/Traveling/Gathering/InCombat/Resting,
MemberIds gồm leader, remove leader→chuyển member[0] deterministic), `Sim/Parties/PartyRegistry.cs` (ctor inject
DiscipleRegistry, SetBus, Create/Add/Remove/Disband, party rỗng→auto-disband, 1 đệ tử 1 party), `Tests/Editor/M13PartyTest.cs`
(menu "Run M13 Tests", 9 test gồm test KHÔNG-DRIFT). Sửa: `Disciple.cs` (+`_skills`/Skills, +`CurrentPartyId`/IsInParty/
**IsAvailableForWork**, +LearnSkill/UnlearnSkill/TrainSkill/JoinParty/LeaveParty; JoinParty clear CurrentBuildingId),
`SampleEvents.cs` (+PartyCreated/Disbanded/MemberAdded/MemberRemoved Event, GlobalBus), `LocalMapManager.cs`
(+PartyRegistry property+init+tick 1b OnDayPassed). ⚠️ `WorkAssignment` là tracker id-only, CHƯA có AI scheduler gọi
TryAutoAssign/Assign (chỉ ở README) → enforce "in-party→ngừng job" là CONTRACT (scheduler tương lai check IsAvailableForWork),
chưa inject DiscipleRegistry vào WorkAssignment. Modding: bỏ 6 core stats (lớp tu tiên) → nạp Base StatBlock thẳng từ JSON,
tầng modifier giữ nguyên (seam = điểm Base).

**CHARACTER UI cho local map — ĐÃ CODE (2026-06-27, chưa wire Editor):** chốt với user: **Cả hai**
(thanh HUD đáy + mở rộng menu), bars = **HP/Linh Lực/Hunger** (BỎ Stress, không trauma-sim), nút F/G/H =
**stub bắn event**. ⚠️ Menu nhân vật full-screen ĐÃ TỒN TẠI (`CharacterMenuController`+`CharacterMenuTabBar`+
`CharacterInfoUI`+`PlayerStatsPage`+ItemScreen) = cửa sổ trên trong ảnh ref → MỞ RỘNG chứ đừng dựng lại.
`EquipmentSlot` enum ĐÃ ĐỦ 9 ô → "9 ô" chỉ là wire prefab. `StatType.MaxLingLi` có; current pool = `UnitMana` (combat).
⚠️ CHỐT LẠI bố cục (user sửa): Status/Skill/Inventory = **MINI character menu DOCK Ở ĐÁY ngay trên PawnBar**,
tab đổi page TRỰC TIẾP (CharacterMenuTabBar vốn đã thế), KHÔNG phải toggle mở menu full-screen. `CharacterMenuController`
full-screen+sidebar GIỮ làm bản chi tiết tùy chọn, KHÔNG dùng cho local map. Binder mini = `UI/Sim/Pawn/PawnInspectorController.cs`
(Bind(d) 1 phát cho pawnBar + _infoUI + _statusPage + _skillPanel + _itemScreen; _autoBindPlayer; _root ẩn khi không chọn pawn).
Files MỚI: `UI/Sim/Pawn/PawnBarController.cs` (thanh đáy: portrait+name+status line+3 bar fillAmount+F/G/H publish
GlobalEventBus, _autoBindPlayer, Bind(d)/SetStatusText), `UI/Sim/Skills/SkillListPanel.cs`+`SkillRowUI.cs` (list
`Disciple.Skills`/LearnedSkill — KHÁC CongPhapPanel=catalog), `UI/Sim/TraitTooltip.cs`+`TraitTooltipTrigger.cs`
(hover thẻ kiểu Exiled Warrior, IPointerEnter/Exit, rich-text modifier xanh/đỏ).
⚠️ REDESIGN MINI MENU (user vẽ lại scene WildLocal, 2026-06-27) — 2 TẦNG TAB: NGOÀI (SheetTabBar) **Status | Skill |
Inventory**; TRONG status (StatusheetButtonContainer) **Traits | Stats | Others(WIP)**. **SKILL TÁCH RA tab ngoài**, KHÔNG
còn trong Status (SkillListPanel + SkillTooltip giữ nguyên, chỉ đổi vị trí). Thêm `UI/Sim/Skills/SkillTooltip.cs`+Trigger
(hover skill→sát thương/linh lực/hồi chiêu/passive theo mastery). Tab Traits = `UI/Sim/TraitsPage.cs` (Main/Sub split bằng
isHidden — false=bẩm sinh/Main, true=unlock-khi-chơi/Sub). **Tab Stats MINI = `UI/Sim/Pawn/PawnStatsPage.cs` (MỚI, GỌN: list
phẳng tên+giá trị + StatTooltip breakdown nguồn) — KHÔNG dùng PlayerStatsPage (đó cho CharacterMenu LỚN: lục giác+traits).**
StatTooltip (`UI/Sim/Stats/StatTooltip.cs`+Trigger) hiện Current+Base+nguồn (Cơ Bản/Trang Bị/Cảnh Giới/Công Pháp/Thiên Phú,
Hom dưới nhãn "Cảnh Giới"); đọc `CharacterStats.ModifiersFor(stat)` (MỚI). DiscipleUIHelper +StatDescription +ModifierSourceLabel
+SkillTypeName. PawnInspectorController: _statsPage giờ là PawnStatsPage (bỏ PlayerStatsPage), skill là tab ngoài riêng.
BUG ĐÃ SỬA: nếu chỉ kéo _pawnBar mà quên kéo _statsPage/_traitsPage/_skillPanel → Bind bỏ qua page → page tự fallback
`_bound ?? PlayerManager.PlayerDisciple` = đệ tử[0] (vd "Cam") → chọn pawn khác chỉ PawnBar đổi, page kẹt ở Cam. FIX:
PawnInspectorController.ResolveRefs() (Awake+Start) tự GetComponentInChildren(true) lấp các page ref nếu null.
**Header CharacterInfoUI BỎ khỏi mini menu** (user chốt) — Name/Realm/Element gộp THẲNG vào PawnStatsPage (_nameLabel/_realmLabel/
_elementLabel tùy chọn); portrait đã ở PawnBar; 4 mini stat nằm trong list. PawnInspectorController bỏ field _infoUI.
**2 tab bar = `CharacterMenuTabBar` (có sẵn) ×2, KHÔNG script mới**: 1 trên SheetTabBar (ngoài Status/Skill/Inventory), 1 trên
StatusheetButtonContainer (trong Traits/Stats/Others). CharacterInfoUI giữ cho CharacterMenu lớn, đừng xoá.
**Wire nút: theo [[feedback-ui-button-wiring]]** — nút tab điền mảng Tabs (Button+Page) của CharacterMenuTabBar (nó AddListener nội
bộ, ĐỪNG thêm OnClick); nút F/G/H wire OnClick Inspector → public method `PawnBarController.OnEnterCombatButton/OnControlToggleButton/
OnRestButton` (đã BỎ AddListener + field Button khỏi PawnBarController cho khớp convention).
⚠️ **WILD MAP CHƯA CÓ UNIT/DISCIPLE THẬT** — `WildPlayerToken` chỉ là sprite click-to-move, KHÔNG gắn Disciple; WildSceneBootstrap chỉ
sinh map. PlayerManager.PlayerDisciple chỉ set khi vào từ WorldMap (SaveData). ĐỂ TEST mini menu: `World/Wild/WildTestPlayerBootstrap.cs`
(DEBUG, kéo lên GameObject trong WildLocal) tạo Disciple giả (stats+realm+Hom+trait+skill) + đăng ký PlayerManager ở Awake → autoBindPlayer
tự bám; có player thật thì tự bỏ qua. Thêm `PlayerManager.SetPlayerForDebug(Disciple)`. PAWN THẬT (player) ĐÃ LÀM: `WildPlayerToken.Unit` (Disciple, Init tự lấy ServiceLocator PlayerManager.PlayerDisciple) +
`SetUnit()`. `WildSceneInteraction` thêm field `_pawnInspector` + branch left-click: click ĐÚNG ô token (token.Cell) →
`_pawnInspector.Bind(token.Unit)` (chọn); ô khác → MoveTo (như cũ). Wire: kéo PawnInspectorController vào WildSceneInteraction._pawnInspector.
MULTI-PAWN SELECT ĐÃ LÀM: WildSceneInteraction left-click dùng `PawnAt(cell)` (FindObjectsByType<WildPlayerToken> tìm token có
Unit ở ô) → chọn pawn đó (bind) + set _selected; ô trống → _selected.MoveTo. NAME TAG: WildPlayerToken tạo `TextMeshPro` 3D GO
riêng (không bị token scale) nổi trên đầu (Unit.Name), follow trong Update, font=TMP_Settings.defaultFontAsset (chỉnh _labelHeight/
_labelFontSize/_labelScale nếu to/nhỏ). WildPlayerToken thêm `Configure(unit,color)`. Test 4 pawn: `World/Wild/WildPartyTestSpawner.cs`
(DEBUG, dùng THAY WildTestPlayerBootstrap) spawn 4 Disciple giả màu cam/vàng/đen/lam quanh EntryPoint, đăng ký [0] làm PlayerManager.
⚠️ CHƯA: beast = BeastActor/CombatUnitDefinition (KHÔNG Disciple) → không bind character menu, cần adapter; SetStatusText("Moving…") chưa nối;
chưa có viền "đã chọn" trên token (selection highlight).
TRAITS DATA: trước giờ **KHÔNG có file trait JSON nào** (thư mục `StreamingAssets/Data/traits/` chưa tồn tại) → tab Traits rỗng.
Đã tạo 4: `trait_kien_cuong`(MaxHP+10%), `trait_luc_si`(ATK+12/DEF+6%), `trait_lanh_loi`(SPD+8%/Evasion+0.05), `trait_sat_tinh`
(isHidden=true→Sub, CritRate+0.05/ATK+5%). Format 1-object/file giống skills. INVENTORY UI: **đã xây ĐẦY ĐỦ sẵn** — `InventoryPanel`
(9 EquipmentSlotUI paperdoll + grid InventorySlotUI + item detail + equip/unequip), dùng hệ **Crafting ItemInstance/ItemSlot/
EquipmentManager** (KHÁC Equipment.Inventory trên Disciple — 2 hệ song song). KHÔNG viết mới, chỉ wire. Mini menu bọc bằng
`UI/Sim/Pawn/PawnInventoryPage.cs` (MỚI — lưu ctx, chỉ Show khi tab active, tránh InventoryPanel.Show tự SetActive đè tab khác);
PawnInspectorController thêm _pawnInventory (auto-resolve, bind ctx). ⚠️ Đệ tử chưa có item nào → slot+grid TRỐNG (chưa có loot/craft
cấp đồ + chưa có per-disciple bag thật, EquipBag truyền `new List<ItemInstance>()` rỗng). Muốn thấy đồ test → cần EquipmentDefinition JSON
+ cấp ItemInstance (bước sau).
INTERACTION (gather/đập): `WildGatherSystem.Begin(f, verb, actor=null)` giờ nhận **actor = pawn đang chọn** (WildSceneInteraction truyền
_selected); fallback _token. CHỐT KIẾN TRÚC (user hỏi có nên làm "interaction global cho mod" không): **KHÔNG làm framework bây giờ** —
hệ reusable/mod ĐÃ CÓ = `TileInteractableScanner`+providers (Register provider mới = điểm cắm mod); mới 1 action (gather) → trừu tượng hóa
sớm = over-engineer. Chỉ gom `IPawnAction` (verb→action, actor+target) khi có ≥2-3 action thật (hái/đập/đánh/nói). (Giả định 1 Assembly-CSharp — UI.Sim↔World đã ref chéo sẵn qua PlayerStatsPage→World.City.UI.) Sửa: `Disciple.cs` (+vitals
CurrentHP/CurrentLingLi/Hunger — bể tách trần, ModifyHP/LingLi/Hunger/RestoreFullVitals, Hunger -5/ngày KHÔNG hệ quả),
`SampleEvents.cs` (+3 PawnEnterCombat/ControlToggle/RestToggle RequestedEvent), `DiscipleUIHelper.cs` (+resist/LingLi
display names, StatDisplayName(string) overload, FormatModifierValue, ModifierColor), `CharacterMenuController.cs`
(+_skillPanel bind), `PlayerStatsPage.cs` (+_traitTooltip gắn trigger runtime lên trait card). ⚠️ BẪY: `EventBus.Publish<T>`
dispatch theo `typeof(T)` → PHẢI publish type cụ thể, đừng truyền base GameEvent. Guide wiring:
`Nizhangjie/M13_CHARACTER_UI_GUIDE.md`. NGOÀI phạm vi: hệ chọn pawn click, listener F/G/H, derive MaxLingLi theo Realm,
icon skill thật, kho "ném hết".

**HOM SCALE — thang đo sức mạnh TOÀN CỤC (ĐÃ CODE 2026-06-27):** phàm nhân=1.0, số liên tục biểu thị "mạnh
gấp mấy lần phàm nhân" (enum cảnh giới rời rạc không balance được). `Stats/HomScale.cs` (static, bảng `_bands`
entry/peak mỗi realm — sửa 1 chỗ để balance; JSON-hóa sau). Curve: Phàm1 / LuyệnKhí 3→5 / TrúcCơ 5→10 / KimĐan
10→20 / ×2-peak mỗi realm tiếp. Công thức `lerp(entry,peak,tierFraction)`; tierFraction = (QiLayer-1)/8 cho Luyện
Khí (CODE 9 TẦNG không phải 10 như spec — endpoint vẫn 3→5), (subTier-1)/3 cho realm trên (4 sub-tier). Cắm balance
qua source **"realm"** (StatModifier dự trù sẵn, trước giờ CHƯA AI DÙNG — cảnh giới chưa từng nhân stat):
`HomScale.ApplyAsStatMultiplier` → PercentMul=(hom-1) lên ATK/DEF/MaxHP/MaxLingLi (KHÔNG crit/kháng/SPD). Tiện:
`disciple.Hom` (đọc, pure function không lưu), `disciple.ApplyHom()`. ĐÃ AUTO-WIRE: DiscipleFactory.Create (cuối) +
PlayerCharacterFactory (lúc tạo, kèm RestoreFullVitals) + HeavenlyTribulation.PublishBreakthrough (đột phá thành công,
re-apply+hồi đầy) + path rớt cảnh giới (ApplyHom, không hồi máu). VD: Căn cốt50→MaxHP base 200, Hom3→600, Trúc Cơ Hom5→1000.
⚠️ Tăng QiLayer (1→9) đổi Hom nhưng CHƯA có site tăng QiLayer (hệ tu luyện XP→tầng chưa code) → khi làm nhớ gọi ApplyHom. ⚠️ HOM = CHỈ SỐ ẨN chỉ để cân bằng, user CHƯA chốt có lên UI hay không → mặc định KHÔNG hiện.
`UI/Sim/HomGaugeUI.cs` giữ sẵn nhưng TÙY CHỌN; `PawnBarController._homGauge` để TRỐNG (null-safe no-op). Test `Tests/Editor/MHomScaleTest.cs` (menu Run M13 Hom Tests). Doc
`Nizhangjie/M13_HOM_SCALE_DESIGN.md`. (Tên "home's" của user → code dùng `Hom`/`HomScale`, đổi caption UI tùy.)

Liên quan: [[project-localmap-wildgen]], [[project-combat-redesign]], [[project-nizhangjie]], [[feedback-unity-layout-group]].
