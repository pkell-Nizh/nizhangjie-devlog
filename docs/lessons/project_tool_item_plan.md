---
name: project_tool_item_plan
description: "⏳ PLAN hệ tool/vật phẩm kiểu roleplay (2026-10-16) — cầm=dùng, bộ điều phối item→thao tác→đối tượng, panel Gadget học Curios; CHƯA CODE, còn câu hỏi mở"
metadata:
  node_type: memory
  type: project
  originSessionId: 2d7a7377-6298-4848-a155-b38ff6f83246
  modified: 2026-10-16T16:02:58.698Z
---

Plan 2026-10-16, CHƯA code. User muốn sửa hệ hook tool vì game chuyển từ colony sim sang roleplay: cách hiện tại (công cụ = món mặc ở `core:weapon`, `HarvestOps.HeldTool`; hotbar đoán loại id theo chuỗi if trong `HotbarItems`; mỗi thao tác một đường riêng) không hợp.

**Hướng đã đồng ý (user chưa bác):** theo MC/Hytale — cầm là dùng, không có ô vũ khí trên UI.
- P0 UI/UX trước (user yêu cầu): hotbar 9 ô = tay chính, ô tay phụ; gợi ý Trái/Phải dưới tâm ngắm lấy từ bộ điều phối; thanh độ bền; inventory kiểu MC + nút **Gadget** mở panel ô phụ (học Curios).
- P1 `HeldItem` + `ItemUseDispatcher` (pawn, món cầm, nút, đối tượng) → handler đăng ký tường minh; món khai `uses[]`; khối khai yêu cầu bằng tag/`harvest_config`. Harvest + ném bóng thành handler, test cũ phải xanh. `toolKind/toolTier` cũ tự quy đổi; save cũ `core:weapon` = món cầm. Kiểm capability gen.
- P2 hotbar chọn = cầm, bỏ chuỗi đoán id, hotbar vào save. P3 handler mới: đặt khối, đổ xô, đuốc, ăn (giữ chuột). P4 ô trang bị học Curios: `uiGroup`, gộp ô nhiều pack (`size`+`operation`), thêm/bớt ô bằng chỉ số/gen, ô theo loài, SlotContext cho `EquipmentFunctionRegistry`, ô cosmetic + ẩn/hiện, `dropRule`, `stackRule`. Khung ô ở CORE (đã có từ M34), nội dung ô ở core pack data — KHÔNG làm mod thư viện kiểu Curios (Curios tồn tại chỉ vì vanilla viết cứng ô).
- P5 chế tác ra trang bị: `WildRecipeDefinition` thêm `outputEquipmentId`. P6 độ bền trên `ItemInstance`/`ItemSaveData` (vật phẩm thường trong ItemLedger không giữ được độ bền). P7 NPC chọn công cụ qua cùng dispatcher. P8 sửa/nâng cấp.

**Thêm 2026-10-16 — tooltip kiểu Jade (MC):** bảng nhỏ đầu màn hình cho thứ đang nhìn = `LookInfo` registry provider (khối/feature/pawn/vật thể, id duy nhất, bật tắt trong config, Shift = chi tiết). Provider lõi: tên+pack, icon công cụ đúng/thiếu bậc (đọc CÙNG dispatcher P1 — không viết luật lần 2), tiến độ đập, máu/tâm trạng pawn, chủ sở hữu (ItemLedger), đồ trong rương (CHỈ khi đã mở/biết — chống lộ loot), cây lớn/ôi thiu/nhiệt. Làm sau P1 vì cần dispatcher; cập nhật theo nhịp (~4 lần/giây), không mỗi khung.

**Đề xuất user 2026-10-16 — bỏ bậc, dùng lực phá:** công cụ khai tag (pickaxe/axe…) + destructive power số cứng; lực ≥ độ cứng khối thì phá được, lực ảnh hưởng thẳng tốc độ; món lai = khai nhiều tag. Em ủng hộ, góp ý: lực THEO TỪNG TAG (`power: {pickaxe:10, axe:6}`) để món lai không thành vua; khối khai `toughness` (một số) + tag; công thức thời gian = base×toughness/lực (có sàn/trần); tay không = tag `hand` lực nhỏ; sai tag = lực tay không; buff/gen/chất lượng cộng/nhân vào lực. Quy đổi tier cũ → lực trong loader. USER CHỐT 2026-10-16: lực < độ cứng ⇒ vẫn phá được nhưng RẤT CHẬM và KHÔNG RƠI ĐỒ, tỉ lệ theo MC (MC: đủ công cụ chia 30, thiếu chia 100 ⇒ chậm ×10/3). Khối "không phá được" (đá nền) = toughness 999999, KHÔNG thêm cờ unbreakable. Công thức đề xuất: giây = k × toughness / lực × (đủ ? 1 : 10/3), k chỉnh trong harvest_config; < 1 tick ⇒ phá tức thì; giữ nhịp chờ ~0,3s giữa 2 khối như MC. Lực theo từng tag — chưa chốt rõ, em mặc định là có.

**Thao tác gắn theo TAG, không theo món (2026-10-16, đề xuất, user hỏi vụ rìu xẻ gỗ chuột phải + món lai):** file `Data/tool_actions/*.json` {nút, cần tag + lực tối thiểu, đối tượng theo tag khối, handler, priority}; dispatcher gom thao tác của món (`uses`) + thao tác mà tag của món được cấp ⇒ món lai có tag axe tự xẻ được, modder không phải biết món lai. Giống Forge ToolActions / NeoForge ItemAbility (AXE_STRIP) — sinh ra đúng vì bài toán paxel. Món tự chặn được bằng `denyActions`. Trùng nút + cùng đối tượng ⇒ priority, rồi lực cao hơn.

**TIẾN ĐỘ 2026-10-16 (đã commit):** P1 phần lõi XONG + VERIFY PLAY MODE.
- Lực phá: `HarvestOps.BreakRuleFor/PowerFor/Evaluate` (Sim/Harvest). Khối: `breakTag`/`toughness`/`tags`; món: `power[{tag,value}]`/`uses`/`denyActions`. Số trong `harvest_config.json`: k=1.5, tay 2, cuốc/rìu đá 7, sắt 10; độ cứng suy = breakWorkload × toughnessPerWorkload (ore 3, tree 1.33, khác 2). Đá 6, quặng sắt 6.6 (cuốc đá được), vàng 8.4 (cần sắt), obsidian 12 (chưa ai đủ). Tier cũ → 4+3×tier.
- Đập khối của người chơi (`WildGatherSystem` block path) dùng Evaluate: không còn cổng chặn, thiếu lực chậm ×10/3 + `Break(..., dropItem:false)`; nghỉ 0.3 s giữa 2 khối. `CanBreak` chỉ còn chặn chất lỏng (đá đáy = 999999, z=0 vẫn chặn "Đáy thế giới").
- `World/Wild/ItemUse/ItemUseDispatcher` + `CoreItemUseHandlers` (break_block, open_block_building, toggle_door, bucket, place_block) + `Data/tool_actions/*.json` (priority giữ đúng thứ tự cũ). `VoxelGameView` chuột trái/phải + gợi ý "[Trái] Mine [Phải] Place" (đỏ khi thiếu lực) đều qua dispatcher.
- Test: `Run ItemUse Tests` 17, `Run Harvest Tests` 27 (thêm 13 ca lực phá); hồi quy Block 80, Shape 46, Tick 28, PlayerActions 31, Decoupling 42 xanh.
- ⚠ Bẫy verify: đá nền granite rơi `stone_rubble` chứ không phải `boulder_granite` (itemId của def) — đừng tưởng là bug không rơi đồ.
- CHƯA: feature 2D vẫn luật cũ; AI `JobGiver_BlockWork` vẫn `Check` cũ (P7); công cụ vẫn đọc từ ô `core:weapon` (P2 hotbar chọn=cầm); hotbar 9 ô/tay phụ/thanh bền/inventory+Gadget (P0 UI) chưa làm; ném bóng chưa thành handler; chưa có thao tác theo tag thật nào (xẻ gỗ cần khối gỗ bóc vỏ).

**P2 XONG 2026-10-16 (đã commit, verify Play):** `Sim/Items/HeldItemOps` = MỘT chỗ trả lời "đang cầm gì": avatar ⇒ ô hotbar đang chọn (trang bị phải nằm trong EquipmentBag hoặc đang mặc), pawn khác ⇒ ô `core:weapon`. `HarvestOps.HeldTool` đi qua đó. Hotbar 9 ô (`kb_hotbar_9`), `HotbarItems.Classify` thay chuỗi đoán id, công cụ trên hotbar KHÔNG còn bị "dùng = mặc vào ô vũ khí". Ném cầu = handler `core:throw_capture` (primary, priority 10 > đập khối). Save: `SaveData.hotbarSlots/hotbarSelected/hotbarVersion` + `SaveSystem/HotbarIO` ở 3 chỗ SaveManager; save cũ (version 0) ⇒ `HotbarController.Update` gọi `HeldItemOps.MigrateLegacyWeapon` khi avatar có mặt (chỉ dời CÔNG CỤ, kiếm ở lại ô vũ khí). Test `Run HeldItem Tests` 17; HotbarStateTest sửa fixture 8→9 ô.

**P0 UI XONG 2026-10-17 (đã commit, chụp Play Mode):** tay phụ = `HotbarState.Offhand` (ô thứ 10 giữ ID, KHÔNG phải ô trang bị) + `SwapOffhand`, phím `kb_swap_offhand` key "None" (USER CHỐT: có input nhưng để trống phím; "" bị ConfigProblems bắt nên dùng "None"); lưu `SaveData.hotbarOffhand`; `HeldItemOps.OffhandId`. Độ bền: `ItemInstance.Durability` (-1 = đầy, save cũ cũng đầy) + `EquipmentDefinition.maxDurability` (đá 131, sắt 250) + `Sim/Items/DurabilityOps` (nơi ghi duy nhất; về 0 ⇒ VỠ, gỡ khỏi túi/ô) + `ItemSaveData.durability`; mòn 1 khi đập khối xong bằng đúng tag (`WildGatherSystem.CompleteBlock`, CHƯA verify Play — avatar đứng trong nhà, không teleport). `EquipmentSlotDef.uiGroup` ("main"/"gadget"; jewelry1/2, back, xianxia:artifact = gadget). VoxelPovHud: thanh bền xanh→vàng→đỏ trên ô hotbar/túi/trang bị, ô tay phụ trái hotbar (ẩn khi trống), tên món hiện 1,5 s, túi đồ: ô main + ô tay phụ cạnh hình nhân vật, nút "Gadget" mở panel trái. Test HeldItem 27.

**P3 XONG 2026-10-17 (đã commit, verify Play):** `ToolActionDefinition.hold/holdSeconds` — thao tác GIỮ chuột: `ItemUseDispatcher.Use` (cú bấm) chỉ nuốt, `UseHeld` chạy mỗi khung (VoxelGameView.UpdateHeldUse khi giữ chuột phải). `ResolveWithOffhand`: chuột phải mà tay chính không có thao tác nào ⇒ thử món tay phụ (HeldTool=null, gợi ý ghi "tay phụ"); khối mờ dùng `PlaceItemId()` hỏi cùng dispatcher. Handler mới ở `CoreItemUseHandlers.Use.cs`: `core:eat` (hold 1.6 s thời gian game, nhả >0.2 s mất tiến độ, qua FoodConsumptionOps.Eat; thanh "Eating …" dưới tâm ngắm), `core:wear` (trang bị không phải công cụ ⇒ mặc). Đuốc = place_block sẵn có, giờ đặt được từ tay phụ. Verify: ăn thịt 70 khung ≈1.6 s Đói 30→78, cú bấm đơn không ăn; cúp tay chính + đuốc tay phụ ⇒ đặt đuốc; đội mũ sắt bằng chuột phải. Test ItemUse 25.

**LookInfo XONG 2026-10-17 (đã commit, chụp Play):** `World/Wild/LookInfo/` — `LookInfoRegistry` (provider đăng ký tường minh, Order rồi id, provider hỏng chỉ cảnh báo 1 lần), `LookTarget` mang sẵn `ItemUseContext` để provider hỏi CÙNG dispatcher/HarvestOps, `LookInfoConfig` (`Data/look_info/look_info_config.json`: enabled, refreshPerSecond 4, disabled[]). Provider lõi: core:block_name (tên + pack), core:pawn (máu/tinh thần), core:break_tool (cần gì, đủ/thiếu lực; Shift: lực/độ cứng/giây), core:break_progress, core:growth, core:container (CHỈ kể đồ khi `LookKnowledge.HasOpened` — đánh dấu ở VoxelPovHud.OpenChest + WildSceneInteraction.OpenBlockBuilding; sổ theo phiên, CHƯA vào save), core:actions (Shift: Trái/Phải từ dispatcher). VoxelGameView.RefreshLookInfo 4 lần/giây (timer unscaled, không dùng TickSpread vì là trình diễn) + `AimedPawn` (sổ token tĩnh). Bảng ở đầu màn hình giữa (VoxelPovHud.SetLookInfo). Test `Run LookInfo Tests` 14. CHƯA verify Play: Shift chi tiết, pawn dưới tâm ngắm, rương đã mở.

**Câu hỏi còn mở:** hết bền vỡ hay hỏng-sửa; phím F đổi tay phụ (F đang là "đi tới làm"); ăn giữ chuột; chế tác 2×2 lưới hay danh sách; ba lô ở nhóm main hay gadget; bóng thu phục ở tay phụ?

Liên quan: [[project_harvest_toolkit]], [[project_m34_equipment_slots]], [[project_inventory_slots]], [[project_capture_tool]], [[project_player_basic_actions]].

## P5 bước 1 XONG (2026-10-17, commit 7539930) — chế tác kép lưới ⟂ danh sách
- Bùi chốt làm CẢ HAI: lưới xếp hình kiểu MC + danh sách kiểu Hytale, chung MỘT kiểu dữ liệu `WildRecipeDefinition`.
- Có `pattern` (+`key`, `mirror` mặc định true) = công thức xếp hình; không có = công thức tự do (`inputs`, bao nhiêu loại cũng được). 27 công thức cũ không phải sửa gì.
- Luôn đọc nguyên liệu qua `recipe.Inputs` (với công thức xếp hình thì suy từ hình), không đọc `.inputs`. WildRecipeText, JobGiver_CraftAtStation và FurnaceOps đã chuyển sang.
- `outputEquipmentId` (khai đúng một trong hai output). JobGiver của NPC vẫn BỎ QUA công thức ra trang bị; hiện chỉ người chơi chế được loại này.
- `WildBuildingDefinition.craftGrid`: 0 = chỉ danh sách, 3 = bàn 3×3. Trạm `core:hand` = 2×2 (hằng số trong RecipeMatchOps).
- `Sim/Crafting/RecipeMatchOps`: GridSizeOf, FitsGrid, MatchesGrid (cắt viền + lật gương; công thức tự do khớp đúng số lượng, không dư loại), FindGridMatch (sắp theo id), MaxCrafts, Layout (sổ công thức tự xếp).
- `ConfigProblems` trên recipe bắt ký tự chưa khai trong key, khai 2 output, và vừa có pattern vừa có inputs.
- Test `RecipeMatchTest` 29/29. Bẫy: menu MCP không thấy menu item mới ngay, phải gọi RunAll qua execute_code.
- Còn lại: bước 2 panel danh sách (chế tay trong túi + khối bàn chế tạo), bước 3 lưới UI, bước 4 công thức mẫu.

## P5 bước 2 XONG (2026-10-17, commit 0446375) — danh sách chế tay + bàn chế tạo
- `Sim/Crafting/PlayerCraftOps`: người chơi chế là ra món NGAY. Đây là chỗ duy nhất trừ liệu và trao đồ (ra vật phẩm hoặc trang bị rời). Túi đầy thì hoàn liệu. Bước 3 (lưới) phải gọi qua đây.
- HUD POV (`VoxelPovHud`): panel "Chế tạo" nằm BÊN PHẢI tấm túi, đối xứng với Gadget bên trái.
  - Trái = chế 1, Shift+trái = chế hết. Món chế được xếp lên đầu.
  - Trạm mặc định là `core:hand`. Đóng túi thì trạm về lại tay. Màn rương thì panel ẩn.
- Khối/building/material `crafting_table` (craftGrid 3, không functions).
  - Chuột phải lên khối → `WildSceneInteraction.OpenBlockBuilding` kiểm tra công trình có craftGrid > 0 → `PovControls.OpenCrafting(defId)` → `VoxelPovHud.OpenCrafting`.
- Công thức: `recipe_crafting_table` (xếp hình PP/PP từ wood_plank; chế ở tay, bàn chế tạo, bàn mộc). Thêm `core:hand` + bàn chế tạo vào `recipe_wood_plank`, thêm bàn chế tạo vào `recipe_torch`.
- Đã sửa lỗi P0: dòng Gadget chèn giữa if/else làm màn rương mất lưới.
- Bẫy verify: `BlockBuildingOps.At` trả công trình ĐẦU TIÊN phủ cột; cột cạnh avatar ở WildLocal có sẵn lửa trại. Screenshot qua manage_camera không chụp được UI overlay.
- Chưa làm: rời xa bàn thì chưa tự đóng (rương thì có).

## P5 bước 3 XONG (2026-10-17, commit abeb5a7) — lưới xếp hình
- Panel Chế tạo có lưới N×N (N = GridSizeOf trạm) → ô kết quả. Trạm chỉ có danh sách thì lưới ẩn.
- Ô lưới GHIM id (giống ô hotbar), KHÔNG lấy đồ ra khỏi túi. Đồ chỉ bị trừ khi bấm ô kết quả (PlayerCraftOps).
  - PruneCraftCells bỏ phần ghim vượt số đồ trong túi.
  - Đóng túi thì xoá lưới. Khác MC có chủ ý: không bao giờ mất đồ, không cần trả đồ về khi đóng.
- Cầm món của túi trên con trỏ + trái lên ô = ghim. Bấm lại cùng món / tay trống / chuột phải = bỏ ghim. Trang bị không đặt được.
- Ô kết quả: trái = chế 1, Shift = chế hết.
- Dòng danh sách (sổ công thức): vừa lưới thì bấm thường sẽ tự xếp (chỉ khi đủ liệu); Shift hoặc công thức không vừa lưới thì chế thẳng.
- Đã verify Play Mode: khớp khi đặt lệch góc; dư 1 tấm thì không khớp; chế ván 5→1 và lưới tự bỏ ghim; sổ công thức 2×2 tự xếp 4 ván.
- Còn lại: bước 4 công thức mẫu (cúp, rìu, ba lô...), rời xa bàn tự đóng.

## P5 bước 4 XONG (2026-10-17, commit cd82c4f) — công thức mẫu → P5 HOÀN TẤT
- Thêm material `stick`; `recipe_stick`: P/P ván, ra 4 que; chế ở tay, bàn chế tạo, bàn mộc.
- Cuốc và rìu (đá vụn / sắt): xếp hình 3×3, chỉ chế ở bàn chế tạo, ra trang bị wep_*. Rìu khớp cả bản lật gương.
- `recipe_leather_pack`: công thức tự do 6 món. `recipe_expedition_pack`: 15 món, vượt lưới 3×3 nên chỉ chế từ danh sách. Đây là ca minh hoạ cho modder cần nhiều hơn 9 ô.
- RecipeMatchTest 38: quét data thật, mọi tham chiếu phải tồn tại và mọi công thức xếp hình tự xếp xong phải khớp lại chính nó (để bắt hai công thức trùng hình).
- Đã verify Play Mode: chẻ que bằng tay, xếp cuốc đá trên bàn, ra wep_stone_pickaxe.
- Nợ còn lại: rời xa bàn chưa tự đóng; NPC chưa làm được đơn ra trang bị; nút offhand vẫn để trống phím.

## P4 (ô kiểu Curios) — chốt 2026-10-17: lõi nhỏ, luật lạ đi qua handler
- Bùi đồng ý: số ô và số pack cứ phình; field lõi giữ nhỏ. `stackRule` và `dropRule` làm thành HANDLER mặc định, KHÔNG làm field.
- Bước 1 XONG (commit 6009759):
  - `EquipmentSlotDef` thêm: size, modifies + operation (file vá set/add/remove, chạy theo thứ tự id), aliases[{id, index, legacyEnum}].
  - Khoá CHỖ: "id" là chỗ 0 (giữ nguyên khoá cũ), "id#i" là chỗ i. `EquipSlotRef` / `EquipmentSlotRegistry.Instances` liệt kê từng chỗ.
  - Hàm mới: ResolveKey(any, out explicit), KeyOf, SplitKey, SizeOf, LegacyEnumOf.
  - Đã gộp core:jewelry1/2 thành core:jewelry size 2. Bí danh jewelry1/jewelry2 và enum 5/6 trỏ vào chỗ 0/1.
  - Equip chỉ nói loại ô ⇒ vào chỗ trống đầu tiên. Có "#i" hoặc bí danh ⇒ đúng chỗ đó. "#0" = đích danh chỗ 0.
  - `EquipFromBag` có overload nhận slotKey (dùng khi bấm ô UI).
  - UI cổ điển: index = uiOrder + chỗ. Pack phải chừa đủ uiOrder cho size.
  - Test SlotSizeTest 22. Bẫy: số loại ô trong test thay đổi tuỳ có nạp Pháp Bảo xianxia hay không.
- Bước 2 XONG (10dfb53) — số chỗ THEO TỪNG PAWN: `EquipmentManager.CountFor/InstancesFor`.
  - Số chỗ = size + `tagBonus` (tag thân xác) + `slotBonus` của đồ đang mặc (đồ ở chính loại ô đó không cộng cho nó) + hook `SlotCountModifiers`.
  - `requiresBodyTags` = ô theo loài.
  - Chỗ bị rút ⇒ `EnforceSlotCounts` trả đồ về EquipmentBag. Restore save có lượt thử lại cho chỗ cộng thêm.
  - `ResolveKey` KHÔNG chặn theo size nữa; chặn theo pawn ở Equip.
- Bước 3+4 XONG (890fa8e) — `EquipmentFunctionContext` có SlotKey/SlotDef/SlotIndex.
  - Luật = handler chọn qua `functions` (không thêm field): `IEquipRule` (AllowEquip, StatScale) và `IDeathDropRule`.
  - Bốn luật lõi trong `CoreEquipmentRules`: core:unique, core:diminish (amount, mặc định 0.5), core:keep_on_death, core:destroy_on_death.
  - `ReapplyRuled` đắp lại modifier của món có luật mỗi khi bộ đồ đổi. ⚠ Việc này xoá luôn modifier mà comp cài chung source key.
  - LootCorpse tôn trọng luật khi chết.
  - Vá bug mất đồ: EquipFromBag rút món khỏi túi ngay cả khi Equip từ chối.
  - Ba lô KHÔNG chuyển sang registry: đó là truy vấn sức chứa, không phải sự kiện mặc.
- Bước 5 XONG (5808928):
  - `cosmeticOf` = ô trang trí: nhận đồ của ô thật nhưng không stat/cơ chế/luật/cộng chỗ. Đã ship core:cosmetic_headwear (uiOrder 20, nhóm cosmetic).
  - Ẩn hình: `EquipmentManager.SetHidden/IsHidden`, lưu ở `DiscipleSaveData.hiddenSlots`. Chuột phải lên ô đang có đồ ở Gadget để bật/tắt.
  - CHƯA có lớp vẽ nào vẽ đồ mặc lên nhân vật, nên ẩn hình mới chỉ là dữ liệu + nhãn "(ẩn)".
- P4 XONG. SlotSizeTest 40, M34 58.

- P7 (2026-10-17, commit ở git log): `Sim/Items/ToolChoiceOps` — NPC dùng Evaluate (cùng luật người chơi), chọn món đủ lực nhanh nhất (hoà ⇒ bền hơn), đổi NGAY qua EquipmentBagOps, mòn 1/khối, còn 1 bền ⇒ cất túi (không gãy, chờ P8 sửa), không món đủ lực ⇒ bỏ việc (JobGiver_BlockWork lọc lúc chọn + FailWhen). Test `ToolChoiceTest` 18. ĐÃ VERIFY Play Mode 2026-10-17 (WildLocal, NPC thật): Vàng cầm rìu + cuốc đá 3 bền trong túi ⇒ tự đổi cuốc, đào 2 khối đá, cuốc về 1 rồi cất túi; khối thứ 3 không ai nhận khi không ai có cuốc dùng được; đưa Lam cuốc sắt ⇒ Lam tự cầm, đào xong, 250→249. ⚠ Bẫy verify: 0h–6h cả đội ngủ theo lịch ⇒ phải đặt lịch Work mới thấy làm việc. NPC chưa tự đi lấy công cụ trong kho.
