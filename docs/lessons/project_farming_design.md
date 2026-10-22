---
name: project_farming_design
description: "Thiết kế trồng trọt tổng quát (2026-10-23, bản nháp chưa code) — Nizhangjie/FARMING_DESIGN.md; PlantSpec + factor registry + onMature + đất có tag; chờ user trả lời 4 câu"
metadata:
  node_type: memory
  type: project
  originSessionId: c91a1631-09ae-4c1b-9675-a98583943f4a
  modified: 2026-10-22T20:23:51.748Z
---

User yêu cầu "trồng trọt, cần thiết kế trước để generalize" (2026-10-23). Bản thiết kế: `Nizhangjie/FARMING_DESIGN.md`. CHƯA code.

- Hướng: không dựng hệ mới. Mở rộng khung có sẵn (khối có nấc + `BlockTicks` randomTick + handler registry).
- Cây = khối có object con `plant` (`PlantSpec`); loader dựng spec từ field cũ (`growDays`/`plantOn`/`harvest*`) để file cũ không phải sửa.
- Điều kiện mọc = danh sách factor (`IGrowthFactor` registry): light/sky/soil/temperature/water_adjacent/submerged/crowding/biome. Pack thêm factor (xianxia:lingqi); factor lạ = 1.
- Kết cục nấc cuối = `onMature` registry: stay/tree/grow_up/fruit_beside/spread.
- Đất = khối có `soil` (tag + fertility + độ ẩm); cây khai tag thay vì id. Cuốc `core:till`, phân bón `core:fertilize`.
- AI: `JobGiver_GrowPlant` làm trọn vòng (hái → gieo → cuốc). Gỡ `feat_crop_*` (dữ liệu trùng di sản 2D).

User chốt §11 (2026-10-23):
1. Chunk ngoài tầm: bù tăng trưởng khi nạp lại, trần 10 ngày.
2. Cuốc hay không do từng loài tự khai: khối cây khai `plant.soil` theo tag; hạt/mầm trỏ tới khối cây nó đặt ra.
3. Lõi chỉ dừng lớn; héo chết để pack tự khai.
4. Mưa tưới ruộng: thời tiết khai cờ `wetsSoil`.

Bước 1 XONG 2026-10-23, commit 125c67fd (nhánh worldgen-3d):
- Code: `Data/Definitions/PlantSpec.cs`, `World/Voxel/PlantGrowth.cs`, `IBlockTickWorld.LightAt`/`TemperatureC`, `SupportOk` nhận tag đất.
- Test `Run Plant Growth Tests` 25/25.
- Bẫy: `BlockDefinition` đọc bằng JsonUtility nên object con `plant` không bao giờ null. Hỏi "có phải cây" bằng `def.Plant` (dựa vào tick), đừng kiểm `plant != null`.

Bước 2 XONG 2026-10-23, commit 43833fe6:
- Đất ruộng, cuốc gỗ/đá, độ ẩm, mưa tưới. Code: `World/Voxel/SoilOps.cs`, thao tác `core:till`, `WeatherDefinition.wetsSoil`.
- Test `Run Plant Growth Tests` 42/42.
- Bẫy: chạy test ngay sau `refresh_unity` có thể còn code CŨ mà data đã MỚI (domain reload chưa xong) ⇒ fail giả. Gọi `refresh_unity` lần nữa với `wait_for_ready` rồi mới chạy test.

Bước 3 XONG 2026-10-23, commit f483eb78:
- Bảng rơi khi hái, cây một vụ, liềm, lúa mì / cà rốt / khoai tây. Code: `World/Wild/Building/PlantHarvestOps.cs`.
- Hỏi "đã chín chưa" qua `BlockDefinition.IsRipe(stage)`, đừng đọc `harvestFromStage`.
- Test 55/55.

Bước 4 XONG 2026-10-23, commit c68deb72:
- Mía, bí, nấm, cỏ cao (nguồn hạt), kết cục `grow_up`/`fruit_beside`/`spread`.
- Test 69/69.
- Bẫy: `OnBlockReplaced` trước đây bật mọi cây phía trên khi khối dưới thành không đặc (kể cả chỉ đổi nấc). Nay hỏi `SupportOk`.

Bước 5 XONG 2026-10-23, commit b5c86b23. Còn bước 6 (bù tăng trưởng vùng ngoài tầm, phân bón) và 7 (pack xianxia):
- Pawn làm ruộng trọn vòng, vùng chọn hạt giống, đã gỡ `feat_crop_*`.
- `Zone.SelectedCropId` giờ là id HẠT; save cũ "crop_x" đọc thành "x_seed".
- Bản vẽ công trình ghi thẳng id khối cây được (`StructureStamper` + `VoxelSolidSector`).
- Test JobGiverGrowPlant 22.
- Bẫy Play Mode: vào game lúc 0h nên pawn đang ngủ theo lịch. Muốn thử việc ban ngày thì `GameClock.SetTick(+8h)`.

Bước 6 XONG 2026-10-23, commit 2fa84d71:
- Bù tăng trưởng ô world nạp lại: `Core/Blocks/SectorSeenStore` + `SaveSystem/SectorSeenIO` (3 chỗ SaveManager) + `BlockTicks` OnSectorLeft/Arrived/ProcessCatchUps.
- Phân bón: `MaterialDefinition.fertilizeSteps`, thao tác `core:fertilize`, `BlockTicks.ForceGrowAt`.
- Test 80/80.
- Bẫy (đã sửa): mỗi khối chỉ được gọi ~1,5 lần/ngày. `ChancePer` chặn ở 1 nên growDays < 0,67 vô nghĩa. Handler cây giờ cho lên nhiều nấc một lần gọi.

Bước 7 (pack xianxia: factor linh khí, đất linh, linh thảo, dời Linh Điền Thảo sang mod) HOÃN theo user 2026-10-23: "mod xianxia có dùng đâu, delay xianxia". Mod xianxia đang tắt (`enabled: false`). Linh Điền Thảo vẫn nằm ở dữ liệu lõi. Nếu làm lại thì đề xuất cũ: linh khí bằng noise theo toạ độ (không đọc Cell 2D).

**Why:** user muốn hệ trồng trọt tổng quát trước khi code, để thêm loài/luật bằng data + handler.

**How to apply:** đọc file thiết kế trước khi code trồng trọt. Làm theo thứ tự §10. Liên quan [[project_m21c_growplant]], [[project_block_world]], [[project_harvest_toolkit]], [[project_modding_strategy]].
