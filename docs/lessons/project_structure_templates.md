---
name: project-structure-templates
description: Bản vẽ công trình chung (StructureTemplateDefinition) + xưởng vẽ trong game — quyết định của user và lộ trình 4 bước
metadata:
  node_type: memory
  type: project
  originSessionId: 8df0bbf9-da06-45ab-aa33-fbed4d8fb3e5
  modified: 2026-10-04T06:36:33.801Z
---

Định dạng bản vẽ chung cho modder, user chốt 2026-10-04 (liên quan [[project-city-outskirts]], [[project-structure-stamper]], [[project-m33-fixed-map]]).

**User đã quyết:**
- Trình tạo template là **màn "xưởng vẽ" NGAY TRONG GAME** (không phải cửa sổ Unity Editor): "chỉ khi ta để bản thân vào người dùng, ta mới biết cái gì là tốt nhất".
- Xưởng vẽ là **scene MỚI**. KHÔNG dùng `HouseBuildPrototype` (đó là bộ test của user, cũng không được commit).
- Nhà trong thành ĐI QUA template (công trình đặc trưng dùng bản vẽ; nhà dân vẫn sinh ngẫu nhiên).
- Gộp ba định dạng cũ (bí cảnh FixedMap, tàn tích StructureStamp, khuôn khu đất) về một.
- Mái: tự suy từ vùng kín, NHƯNG phải có thêm khả năng vẽ tay cho modder muốn.
- Nhiều tầng: chỉ chừa chỗ trong định dạng (`TemplateLayer.floor`), chưa làm — user còn rối phần này.
- Tiêu chí của modder sẽ nhiều hơn nhiều so với 3 ví dụ ban đầu ⇒ thiết kế theo viên gạch ghép được, không thêm trường cho từng nhu cầu.

**Why:** modder cần nói được "chỗ này trang trí ngẫu nhiên", "công trình sinh cứng", "đổi 4 phòng với nhau lúc gen"… mà vẽ ASCII tay không chịu nổi quá cỡ ruộng rau.

**How to apply — lộ trình 4 bước:**
1. Định dạng + bộ gen (XONG 2026-10-04, chưa commit lúc viết): `Data/Definitions/StructureTemplateDefinition.cs` (lớp ô, `chance`/`choices`, `regions` + `swapGroup` + `transform`, `stand`/`mark`/`entry`), `World/Wild/Generation/TemplateResolver.cs` (`TemplateResolver.Resolve` thuần + tất định, `StructureTemplates.Get/Raw/Prime/Stamp`), data `structure_templates/` (2 thư mục). `SiteTemplateDefinition` + `site_templates/` đã xoá; ngoại ô dùng bản vẽ mới. Test `Run Structure Template Tests` (32).
2. Xưởng vẽ bản tối thiểu (XONG 2026-10-04, chưa commit lúc viết): scene `TemplateWorkshop` (dựng bằng menu `Nizhangjie/Xưởng vẽ/Dựng lại scene`, sao lưới tilemap + `WildTilemapRenderer` từ WildLocal), `World/Workshop/TemplateDocument.cs` (mô hình thuần: sửa ô, hoàn tác, JSON gọn) + `TemplateWorkshopController.cs` (UI dựng bằng code; xem trước = gen thật). Lưu vào mod `StreamingAssets/Mods/workshop/` hoặc (Editor) game gốc cả 2 thư mục. Vào từ nút "Xưởng Vẽ" ở MainMenu hoặc menu `Nizhangjie/Xưởng vẽ/Mở xưởng vẽ (Play)`. Bẫy: sửa .cs lúc đang Play ⇒ domain reload xoá state của xưởng; viền cỏ quanh bản vẽ phải ≥ 400 ô kẻo `RoomDetector` tô "trong nhà" cả bên ngoài.
3. Nhà trong thành qua template (XONG 2026-10-04, chưa commit lúc viết): `CityBuildingDefinition.templateId` → `World/City/Generation/TemplateBuilding.cs` (ruột nhà từ bản vẽ, mái tự suy = vùng kín + tường quanh, đè bằng `roof: on/off`), `PlacedBuilding.TemplateId/Seed/Variant`; Furnisher + BuildPieceStamper bỏ qua nhà `FromTemplate`, `SettlementStampPass.StampTemplates` dán. Mẫu: `city_care_center` (14×14, có hiên không mái). Nợ: NPC đứng ở hiên vẫn bị coi là "trong nhà" (ẩn tới khi mở nóc); mỗi công trình mới 1 bản vẽ (chưa có danh sách trọng số); khuôn ngoại ô vẫn dùng field riêng `siteTemplateId`.
4. Tag + gộp định dạng (XONG 2026-10-04, chưa commit lúc viết). User yêu cầu thêm: "bộ tag, tag cho công trình, tag cho ô ngẫu nhiên (có chọn specify)".
   - Định dạng: `tags` + `weight` trên bản vẽ; ô `slot` (`TemplateSlot`: kind building/feature/item, `tags`/`anyTags`/`notTags`, `ids` = chỉ định đích danh, `maxWidth/Height`); option của vùng bốc theo tag; `when` (ConditionRef) trên ô / mục bốc / option — facts null = coi như đạt; khối `wild` (rải hoang dã) và `map` (bí cảnh).
   - Bộ gen nhận `TemplateCatalog` (`StructureTemplates.Catalog()`); tag đồ đạc = `tags` + `ctx:{suitableContexts}`, cây/vật và vật phẩm = `category`.
   - Nơi dùng: `CityBuildingDefinition.templateTags` / `siteTemplateTags` — các bản ĐƯỢC khác cỡ (user yêu cầu bỏ giới hạn cùng cỡ): bộ xếp nhà gọi `TemplateBuilding.SizedFor` bốc bản vẽ cho từng căn TRƯỚC khi chiếm lô, `TryTemplate` chỉ dựng bản khít lô; thành chủ (LandmarkPlacer) chưa đi qua đường này. Trong một thành, các căn cùng loại ưu tiên cỡ/bản vẽ CHƯA dùng (`Context.TemplateSizeUse`, `TemplateBuilding.LeastUsed`) — user chọn hướng "tăng số lượng + không lặp" trước; "cỡ theo cấp thành/vị trí" để sau. Mẫu: 3 bản Care Center (2 bản 14×14, 1 bản 10×9), thành phố lớn 2–4 căn. Thành phố và ngoại ô CHƯA truyền facts cho `when`.
   - Gộp: `structure_stamps/` và `fixed_maps/` của game gốc đã xoá, dữ liệu nằm ở `structure_templates/`. `StructureStampDefinition` / `FixedMapDefinition` còn lại làm "phiếu" bọc bản vẽ (`template`), file kiểu cũ của mod vẫn nạp (DevPokemon, tensura đang dùng). `pool` vẫn do `StructureStampPass` bốc sau khi gen.
   - Xưởng: bảng phải 3 trang (Bản vẽ / Ngẫu nhiên / Vùng), công cụ Vùng (G), Alt+chuột trái lấy mẫu. Chưa có bảng sửa `wild`/`map`/`choices` có trọng số/`when` ngoài biome.
   - Test `Run Structure Template Tests` 118.
5. Bảng "Công trình" trong xưởng vẽ (bản đầu XONG 2026-10-04, chưa commit lúc viết) — user chốt: quản lý TOÀN BỘ công trình (trong thành, ngoại ô, hoang dã, bí cảnh), CHỈ sửa phần bản vẽ + số lượng, chưa cần dựng thử cả thành ("tinh chỉnh sau"). `World/Workshop/StructureManagerModel.cs` (thuần: gom dòng, lý do bản vẽ bị loại, vá JSON giữ nguyên trường khác) + `StructureManagerPanel.cs` (UI 3 cột, nút "Công trình" trên thanh trên). Lưu theo công tắc mod/game gốc của xưởng.
