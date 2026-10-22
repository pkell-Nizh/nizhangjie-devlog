# Audit Spaghetti — Nizhangjie (bắt đầu 2026-07-09)

Mục đích: đọc TOÀN BỘ ~61k dòng code game trong `Nizhangjie/Assets/Scripts` để đánh giá spaghetti.
File này là note tiến độ — nếu session đứt giữa chừng, đọc file này để biết đã đọc tới đâu.

## Tiến độ đọc
- [x] Core (15 file, 1.7k dòng) — ĐỌC XONG, sạch
- [x] Stats (9 file, 0.5k) — ĐỌC XONG, sạch
- [x] Data (32 file, 4.4k) — ĐỌC XONG
- [x] SaveSystem / Equipment / Modding / Localization (11 file, 1.2k) — ĐỌC XONG
- [x] Sim (57 file, 8.8k) — ĐỌC XONG (Quests/Parties/World/Items/Player cũng sạch; Quest dùng builder pattern gọn; PartyRegistry orchestrate invariant tốt; GlobalWarehouse/PlayerItemBag/DiscipleRegistry đều có static Get() get-or-create qua ServiceLocator — nhất quán nội bộ nhưng là service-locator-as-singleton, xem NỢ #1)
- [x] LocalMap (27 file, 3.1k) — ĐỌC XONG
- [ ] World (124 file, 16.1k)
- [ ] Combat (68 file, 9.3k)
- [ ] UI (56 file, 7.7k)
- [ ] Tests (29 file, 5.5k — chỉ skim cấu trúc)
- Bỏ qua: ThirdParty/FastNoiseLite.cs, AstarPathfindingProject, TextMesh Pro, Plugins, _LegacyCombat (legacy đã biết)

## Nghi vấn ban đầu (chưa xác nhận, thấy từ danh sách file)
- File 2 dòng kiểu shim: `Sim/Disciple.cs`, `Sim/DiscipleRegistry.cs`, `Sim/DiscipleFactory.cs`, `Data/TraitDefinition.cs`, `Data/TraitCategory.cs`, `Data/RealmDefinition.cs`, `Stats/StatId.cs`, `Stats/StatBlock.cs` — cần xem là type-forward hay rác.
- Trùng tên 2 nơi: `PlayerCreationConfig` (SaveSystem 112d + UI/MainMenu 50d), `StatRowUI` (UI/MainMenu 121d + UI/Sim/Stats 35d).
- `Core/Events/SampleEvents.cs` 648 dòng — "Sample" mà to nhất Core?
- `Data/Editor/DataSampleInitializer.cs` 1186 dòng — file to nhất project.
- 2 hệ thời gian: `Core/Time/GameClock` vs `LocalMap/Time/LocalMapClock` + `Combat/RTwP/CombatTickClock` — cần xem quan hệ.
- 2 pathfinder: `LocalMap/Pathfinding/LocalPathfinder` vs `Combat/Movement/GridPathfinder` (+ A* package).

## ✅ ĐÃ XỬ LÝ 4 NỢ (2026-07-09)

Tất cả compile-clean qua grep (không test được Unity ở môi trường này).

1. **2 DiscipleRegistry song song → THỐNG NHẤT.** `LocalMapManager.InitSystems()` đổi `new DiscipleRegistry()`
   → `DiscipleRegistry.Get()` (roster toàn cục qua ServiceLocator, chung với Wild). Get() tự wire bus vì
   InitBuses chạy trước → không double-SetBus. Cập nhật doc trong DiscipleRegistry.cs + M_INVENTORY_FLOW §6 + M_CODEBASE_AUDIT B3.

2. **2 hệ trang bị + 2 enum slot xung đột → GỠ HỆ CHẾT.** Xoá `Equipment/Inventory.cs`, `EquipmentItem.cs`,
   `Enums/EquipmentSlot.cs`, `Enums/ItemRarity.cs` (+ .meta + folder Enums). Sever `Disciple.Equipment` khỏi
   constructor + gỡ `using Nizhangjie.Equipment`. Viết lại `M2DataTest` Test 6 sang `EquipmentManager`+`ItemInstance`+`ItemSlot`.
   Dọn 2 TODO comment trong DiscipleSaveData + comment "ItemRarity" trong EquipmentDefinition. Còn DUY NHẤT hệ `ItemSlot`.

3. **Logic derive/craft trùng lặp → GỘP 1 NGUỒN.** (a) `DiscipleFactory.DeriveCombatStats` → public; PlayerCharacterFactory
   + WildTestPlayerBootstrap gọi lại nó thay vì chép công thức (bản mirror thứ 3 trong debug bootstrap cũng gộp,
   CritRate debug đổi 0.08→0.055 vô hại). (b) `CraftingEngine`: tách lõi roll thành private `RollAndProduce()`,
   `AttemptCraft` + `ResolveCraft` cùng gọi (trước copy-paste ~25 dòng).

4. **LocalMapManager đồng hồ riêng → NỐI GameClock.** Gỡ `DaysPerRealSecond` + `_accumulator` + `Update()`;
   thêm `InitClock()` subscribe `GameClock.OnDayTick += Tick` (fallback tạo GameClock cục bộ giống LocalMapClock),
   unsubscribe ở OnDestroy. `IsPaused` giờ delegate `GameClock.Paused`. ForceTickDay giữ nguyên cho test.
   **Lưu ý pacing:** sect-sim giờ chạy theo nhịp GameClock (mặc định 360s/ngày) thay vì 1 ngày/giây cũ —
   đây là hệ quả CÓ CHỦ ĐÍCH của việc thống nhất 1 đồng hồ; designer chỉnh tốc qua SecondsPerDay/LocalMapClock.
   Không file .cs nào ngoài đọc `DaysPerRealSecond`/`IsPaused` nên an toàn compile.

**Chưa đụng (ngoài phạm vi 4 nợ này, còn tồn):** SaveManager/PlayerCreationConfig singleton (B1), data-loading lai (B2),
UnitDefinitionSO (D4), tombstone files, và toàn bộ World/Combat/UI CHƯA đọc hết.

## Phát hiện (cập nhật dần)

### Core — SẠCH (điểm 9/10)
- EventBus/GameEvent/ScopedBuses/ServiceLocator/GameClock/GameBootstrap: gọn, doc tốt, tách scope rõ, không vòng phụ thuộc. Handler snapshot khi publish (an toàn re-entrancy), try/catch từng handler.
- `SampleEvents.cs` (648d): tên "Sample" sai lệch — thực chất là REGISTRY toàn bộ event M1→M13. Không spaghetti (chỉ là DTO khai báo) nhưng nên đổi tên `GameEvents.cs` + tách theo domain khi >1000 dòng. CLAUDE.md quy định event mới thêm vào đây → file sẽ phình vô hạn. **Mức: nhẹ, cosmetic.**
- Nhược nhỏ: `EventBus.Subscribe` sort toàn list mỗi lần subscribe (O(n log n)) — chấp nhận được với số handler nhỏ.

### Stats — SẠCH (9/10)
- CharacterStats: 2 lớp base+modifier, dirty-flag lazy recalc, công thức Flat→PercentAdd→PercentMul đúng như doc. HomScale bảng band rõ ràng, có source key "realm" tự dọn.
- `StatId.cs`/`StatBlock.cs` = file 2 dòng "đã thay thế" — RÁC tombstone, nên xóa (đã liệt kê trong CLAUDE.md gotchas nên vô hại, nhưng là nợ dọn dẹp).

### Data — SẠCH (9/10)
- JsonDataLoader/DataRegistry/Def/GameDatabase/ModLoader/LocalizationManager: pipeline load→resolve→validate→freeze rõ ràng, mod-aware, ConfigErrors fail-fast. ModLoader topo-sort thuần (testable). Definitions toàn POCO không logic (chỉ helper match nhỏ) — đúng data-driven.
- Tombstone 2 dòng: `Data/RealmDefinition.cs`, `Data/TraitDefinition.cs`, `Data/TraitCategory.cs` — nên xóa.
- `DataSampleInitializer.cs` (1186d): editor seeder, chỉ chuỗi JSON. KHÔNG spaghetti nhưng là NGUỒN SỰ THẬT THỨ 2 trùng với JSON thật trong Assets/Data → nguy cơ drift. Riêng traits dùng `WriteAlways` = GHI ĐÈ file trait đã chỉnh tay mỗi lần chạy menu — bẫy mất dữ liệu nhẹ.
- Nhiều Definition có `DefaultSet()` hardcode mirror JSON (MapFeature/RegionArchetype/RockType/WildBuilding) — chấp nhận được (fallback), nhưng cùng nguy cơ drift, cần kỷ luật sync.
- `UnitDefinitionSO` — SO trái luật JSON-first, đã ghi trong CLAUDE.md gotchas, vẫn tồn tại.

### SaveSystem/Equipment/Modding/Localization — SẠCH, 2 điểm trừ
- **`SaveManager.Instance` + `PlayerCreationConfig.Instance` = static singleton — VI PHẠM luật kiến trúc #2 "No singletons"** (dù có comment biện hộ "safe trước GameBootstrap"). Không gây spaghetti thực tế (SaveManager gọn), nhưng là ngoại lệ luật cần ghi nhận/chuẩn hóa.
- `PlayerCharacterFactory.DeriveCombatStats` TRÙNG LẶP công thức derive với `DiscipleFactory.DeriveCombatStats` (comment tự nhận). Đổi công thức 1 nơi quên nơi kia = bug lệch stat player vs đệ tử. → nên gộp 1 chỗ.
- Equipment/Inventory: sạch, source key `equipment:{InstanceId}` đúng quy ước.
- Lưu ý nhỏ: `EquipmentSlot` enum (Weapon=0..Artifact=8) ≠ chú thích trong `EquipmentDefinition.slot` (ghi 0=Headwear,7=Weapon…) — DOC SAI LỆCH so với enum thật, dễ author nhầm slot JSON. → XEM MỤC "2 HỆ TRANG BỊ" bên dưới, đây là triệu chứng của vấn đề lớn hơn.

### Sim (đọc ~60%, đã xong Disciples/Sect/Economy/Crafting) — KHÁ SẠCH nhưng có 3 cụm nợ đáng kể

**⚠ NỢ #1 — 2 DiscipleRegistry song song (code TỰ NHẬN):**
`DiscipleRegistry.Get()` (static, ServiceLocator, dùng ở scene Wild) vs `LocalMapManager` tự `new DiscipleRegistry()` riêng (scene City/Sect). Doc trong code ghi rõ "nợ kỹ thuật, xem M_INVENTORY_SYSTEM_FLOW §Spaghetti". Đây là mầm spaghetti thực thụ: cùng khái niệm "roster đệ tử" có 2 nguồn sự thật tùy scene.

**⚠ NỢ #2 — 2 HỆ TRANG BỊ song song + 2 enum slot XUNG ĐỘT giá trị:**
- Legacy: `Equipment/Inventory` + `EquipmentItem` + enum `EquipmentSlot` (Weapon=0, Armor=1, Helmet=2, Boots=3, Belt=4, Ring1=5...).
- Canonical: `Sim/Crafting/EquipmentManager` + `ItemInstance` + enum `ItemSlot` (Headwear=0, Armor=1, Leggings=2, Gloves=3, Boots=4... Weapon=7).
- CÙNG field JSON `EquipmentDefinition.slot` (int) nhưng `EquipmentItem.Slot` cast sang `EquipmentSlot` còn EquipmentManager dùng `ItemSlot` → **slot=0 nghĩa là Weapon ở hệ cũ nhưng Headwear ở hệ mới**. `Disciple.Equipment` (Inventory legacy) vẫn được khởi tạo trong constructor + comment dài giải thích "đừng dùng". Cần gỡ hệ legacy dứt điểm.

**⚠ NỢ #3 — logic derive/craft trùng lặp:**
- `DeriveCombatStats` trùng 2 nơi (DiscipleFactory + PlayerCharacterFactory) — đã ghi ở trên.
- `CraftingEngine.AttemptCraft` vs `ResolveCraft`: ~25 dòng roll success/quality/output copy-paste giữa 2 hàm (khác nhau chỉ ở chỗ tiêu nguyên liệu). Sửa công thức 1 nơi dễ quên nơi kia.

**Điểm tốt:** mọi system (Salary/Injury/Aging/TraitUnlock/Relationship/Market/Shop/Crafting/Queue...) đều là plain-C# class nhỏ <350 dòng, 1 trách nhiệm, giao tiếp qua GlobalEventBus + registry, doc đầy đủ. KHÔNG có god-class, KHÔNG vòng phụ thuộc. Đây là kiến trúc RimWorld-style kỷ luật tốt.

**Lưu ý nhỏ Sim:**
- Hệ sim dùng `UnityEngine.Random` không seed (Tribulation/Salary/Shop/Crafting/Discovery/Competitor) — chấp nhận được (luật determinism chỉ áp cho gen), nhưng nếu sau này muốn replay/test sim thì phải đổi.
- `AlchemySystem` comment thú nhận công thức stat "(Comprehension+Agility)/2" trong doc nhưng code chỉ dùng Comprehension — doc/code lệch nhẹ.
- `SalarySystem` doc ghi "Realm 0 = Luyện Khí" nhưng enum Realm 0 = Mortal — comment lệch enum (code vẫn chạy đúng vì chỉ index bảng).

### LocalMap — SẠCH (8.5/10), 1 vấn đề thời gian đáng chú ý

**⚠ NỢ #4 — LocalMapManager có ĐỒNG HỒ RIÊNG, không nối GameClock:**
`LocalMapManager.Update()` tự tích lũy `_accumulator` theo `DaysPerRealSecond` (mặc định 1 ngày/giây!) và giữ `TotalDays` riêng — hoàn toàn TÁCH khỏi chuỗi chính GameClock (360s/ngày) → GameCalendar → LocalMapClock. Tức tồn tại 2 dòng thời gian với 2 nhịp khác nhau nếu cả hai cùng chạy trong 1 scene. CLAUDE.md tuyên bố GameClock là "NGUỒN THỜI GIAN TOÀN CỤC" nhưng orchestrator sect-sim chưa migrate. (LocalMapClock/GameCalendar thì đã làm ĐÚNG facade như doc.)
- Kéo theo: `LocalMapManager` cũng tự `new DiscipleRegistry()` (NỢ #1) — file này là điểm hội tụ của 2 món nợ lớn nhất.

**Điểm tốt LocalMap:** LocalPathfinder A* octile + chống cắt góc chuẩn; IPathfinder+Adapter DI đúng bài; JobManager/JobDefinition/ActiveJob rõ ràng, data-driven, RNG có seed; BuildingEfficiencyTable data-driven kèm fallback legacy có ghi chú; Tile/TileType append-only đúng quy ước; ZoneManager gọn.
- Lưu ý nhỏ: `BuildingController.MapBuildingToResource` vẫn hardcode keyword→resource (comment tự nhận "M5+: load từ JSON") — cùng họ với fallback keyword của EfficiencyTable, chưa data-driven.
- `GridSystem.GetTilesInZone` quét toàn map O(W×H) mỗi lần gọi — chấp nhận được ở 64×64, để ý nếu map 200×200 gọi mỗi tick.
