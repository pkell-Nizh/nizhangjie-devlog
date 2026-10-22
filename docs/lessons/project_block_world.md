---
name: project_block_world
description: "Thế giới bằng khối thật (2026-10-13) — tầng tuyệt đối 0..255 (tầng 64 = mặt đất chuẩn), nền sinh từ seed + BlockStore là SỔ SỬA (khối đặt + chỗ đào = không khí), đập/đặt mọi khối, loại ô tự đồng bộ theo khối"
metadata:
  node_type: memory
  type: project
  originSessionId: b834bf4f-f364-4be2-a429-910f012d4ad7
  modified: 2026-10-12T20:12:17.000Z
---

User chốt 2026-10-13: thế giới bằng khối, sửa được mọi chỗ, thế giới nhớ thay đổi; 2D ĐÓNG KHO; cao 256 khối tính từ đáy. Mình chọn mặc định (user chưa trả lời): cây vẫn là feature (chưa thành khối thân/lá), làm bước 1–3 trước (nền khối + đào/đặt + lưu), tường/đá chưa gộp hẳn.

**Kiến trúc (ĐÃ CODE, test 72/72, ĐÃ VERIFY Play Mode bằng execute_code + ảnh chụp camera, chưa thử bằng phím thật):**
- `Core/Blocks/BlockStore.cs`: `BlockLayers` (Count 256, `Zero`=64: tầng 64 chiếm độ cao voxel [0,1)), `IBlockTerrain` (nền sinh sẵn — `VoxelWorld` cài vào ServiceLocator), `BlockStore` = SỔ SỬA khoá (x,y abs, tầng abs), defId rỗng = đã đào. `BlockDefs.WallId` = tầng tường.
- `World/Voxel/TerrainBlocks` + `Data/voxel/terrain_blocks.json`: loại ô ⇒ mặt/đất(dày)/đá/đá đáy. Khối nền mới trong `Data/blocks` (grass/dirt/sand/stone/bedrock/water/lava/snow/ice/gravel/mud) + vật phẩm `dirt`, `stone_rubble`, `snow_pile`, `ice_chunk`. `BlockDefinition` += `placeable`/`unbreakable`/`surfaceTile`.
- `VoxelSolidSector`: NỀN khối (`BaseRuns`/`BaseBlockAt`, xem bước 1 dưới — phần đã sửa ở chunk). Cột đá = khối rơi ra đá tảng của vách (`BlockDefs.ForYield`); vật phẩm không có khối ⇒ `Ore` (khai thác theo feature như cũ). Nước thấp hơn bờ đúng 1 khối (bỏ 0.55).
- `VoxelWorld`: sàn tìm đường = sàn chunk (khối không leo được ⇒ 1e6); đồng bộ loại ô sau mỗi lần dựng (`DesiredType`: đào hết cỏ ⇒ Dirt; đào cột đá xuống ≤ mặt đất+1 ⇒ Stone đi được + vách cạn; lấp sông bằng đất ⇒ Dirt); loại GỐC giữ ở `_origType` và `VoxelSource` chụp theo loại gốc. Chỉ đổi khối ⇒ dùng lại độ nổi đã tính. Quy đổi save cũ (`ConvertLegacy`).
- `BlockPlacementOps`: `PlacementTarget.Cell.z` = tầng TUYỆT ĐỐI. `TryBlockAt`/`FloorLayers`/`CanBreak`/`Break` (nền ⇒ ghi không khí; khối đặt trên không khí ⇒ bỏ mục; đặt lại đúng khối nền ⇒ bỏ mục). Chặn: đá đáy/nước, khối dưới chân cây/công trình/tường.
- `BlockHeightBridge` ĐÃ XOÁ (VoxelWorld ghi sàn). Save: `SaveData.blockFormat` (1) + `legacyBlocks`.
- Lớp vẽ: `VoxelMapMesher` vẽ theo chunk (khối mặt nền tô màu loại ô, khối khác màu khối); `VoxelGameView` ngắm theo `VoxelWorld.IsSolidLayer`, ngắm mặt sàn dưới cây/bụi hoặc cột quặng ⇒ khai thác feature.

**BƯỚC 1 lộ trình — CHUNK KHỐI (2026-10-13, test 77/77, ĐÃ VERIFY Play Mode):** `World/Voxel/VoxelChunk.cs` (`VoxelChunk` 16×16 cột × 16 lát 16³, lát đồng nhất chỉ giữ 1 số; `VoxelChunkSet` = mọi chunk của một ô world + `Floor`/`TopHeights`/`Blocked`, `ApplyEdit` sửa tại chỗ) + `BlockIds` (ushort ⇄ id chuỗi, chỉ trong RAM). `VoxelSolidSector` giờ CHỈ còn nền + tường + đồ đặt + mái (không áp sổ); `BaseRuns` công khai; cột đá = đá thường dưới (mặt đất −3) + đá vách ở trên. `VoxelWorld`: dựng hình ⇒ dựng chunk ở luồng nền; `BlockStore.Changed` ⇒ `ApplyEdit` + sàn + bậc tìm đường + loại ô NGAY trên luồng chính, bắn `ChunkChanged` (không dựng lại cả hình); `Gather`/`IsSolid` đọc chunk (hầm/khối treo đúng tuyệt đối). `IBlockTerrain.FloorLayers` (mặc định −1) để luật đặt/đập hỏi sàn từ chunk. Mesher: `EmitChunk`/`RebuildChunk` (mặt kề không khí, bỏ lát đặc kín), `VoxelGameView` giữ `ChunkGos[]` và thay mesh đúng chunk. Số đo WildLocal thế giới liền 9 ô world: 1521 chunk, ~25 MB lát, ~2,8 triệu đỉnh; đặt/đập ~3,5 ms/lần (quét `data.Features` để kiểm "vướng cây" chiếm phần lớn), vẽ lại 1 chunk ~0,8 ms. **Nợ:** chưa gộp mặt (greedy) ⇒ nhiều đỉnh; tra feature theo ô còn tuyến tính; mặt ở ranh ô world vẽ cả hai phía (khuất, vô hại).

**BƯỚC 2 — LÒNG ĐẤT 3D (2026-10-13, test 85/85, ĐÃ VERIFY Play Mode + ảnh):** `World/Voxel/WorldGen3D.cs` + `Data/voxel/worldgen_3d.json` (2 bản) + 8 khối quặng (`block_coal/iron/copper/silver/gold/quartz/jade/gem_ore`, không đặt được, đào ra vật phẩm quặng có sẵn). Chạy trong `VoxelChunkSet.Build` sau khi đổ nền, trước sổ sửa: đá sâu (`block_slate` dưới ~tầng 34) → đá lẫn (hoa cương/đá vôi/cẩm thạch) → mạch quặng theo dải độ sâu (RNG gieo theo seed + toạ độ TUYỆT ĐỐI của chunk) → hang "spaghetti" (giao 2 mặt noise 3D, cách mặt 5 khối, miệng hang mở theo noise 2D; núi mở dễ hơn ⇒ vách hõm/mái nhô). Noise 3D tự viết (value noise, KHÔNG `Mathf.PerlinNoise`), lưới nút 4×4×4 căn toạ độ tuyệt đối ⇒ khớp mép giữa ô world (test lệch 0). Ô tường/nước/mái/đường/sàn nhà/đồ đặt không bị đục mặt. `VoxelChunkSet` nhớ khối sinh sẵn DƯỚI mỗi ô đã sửa (`BaseAt`) ⇒ bỏ mục trả đúng quặng/hang; `VoxelWorld.TryBaseBlock` đọc từ đó. **Luật sàn đổi:** sàn = đỉnh đoạn đặc cao nhất BẮT ĐẦU dưới mặt nền gốc ⇒ hang/hầm dưới đất KHÔNG làm mặt đất thành hố (NPC chưa vào hang được — bước 3). Lớp vẽ: mặt khối tách "lộ trời" (`SkyTop` mỗi cột) ⟂ "trong hang"; mặt hang chỉ dựng cho chunk trong 56 khối quanh camera (`UpdateCaveMeshes`, `BuildCaveMesh`) ⇒ 9 ô world: 14,5 triệu đỉnh → 2,87 triệu + 0,44 triệu. Số đo: dựng lòng đất 1 ô world 200² ~394 ms ở luồng nền (sau khi tối ưu nội suy theo cột, trước đó 1,3 s); lát cấp ~59 MB. **Nợ:** đưa sang Burst; hang chưa có ánh sáng (sáng như ngoài trời — bước 6).

**BƯỚC 3 — TÌM ĐƯỜNG NHIỀU SÀN (2026-10-13, test Multi-Floor 45/45, ĐÃ VERIFY Play Mode + ảnh):** chi tiết ở [[project_multifloor_pathfinding]]. Tóm tắt: ô lưới = sàn chính (z=0), hang/cầu/tầng lầu = chỗ đứng phụ (`PathFloors`, z = tầng tuyệt đối); `VoxelChunkSet.ColumnSpots` tính, `VoxelWorld` đổ vào.

**Bẫy:** Editor mất focus ⇒ `Update` không chạy ⇒ sửa khối không dựng lại; verify dùng `BuildAllNow` (nay dựng cả hình bẩn) hoặc gọi `Update` qua reflection. Lớp 3D tắt sẵn khi vào Play (F9 / `TurnOn`).

**Còn lại:** cây thành khối; tường/cửa thành khối thật; chất lỏng thật (nước đang là khối đặc không phá được); đập khối hàng loạt cho NPC (job). Thứ tự làm tiếp: [[project_world_like_minecraft_roadmap]].

Liên quan: [[project_block_placement]], [[project_voxel_world_owned_by_game]], [[project-render-2d3d-coupling]], [[project-kinematic-body]].
