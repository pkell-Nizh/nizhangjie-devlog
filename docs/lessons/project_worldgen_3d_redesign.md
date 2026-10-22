---
name: project-worldgen-3d-redesign
description: "Viết lại bộ gen cho thế giới 3D liền (2026-10-19) — quyết định user đã chốt: trục đứng không giới hạn, khung gen 0..512, sông chảy thật, world+spline+hồ sơ biome; trạng thái từng giai đoạn"
metadata:
  node_type: memory
  type: project
  originSessionId: ccdc1c17-b874-41b3-8212-253b84f54342
  modified: 2026-10-18T19:59:34.918Z
---

User 2026-10-19: bộ gen cũ (2D colonist) lỗi thời, cần viết lại cho 3D liền. Đã bàn, tham khảo Minecraft 1.18 (spline continentalness/erosion/peaks-valleys, sông = rãnh tới mực biển) + Hytale (zone → biome tự khai hồ sơ địa hình, trộn mép, lớp/phủ/prefab là data).

**Hiện trạng tìm ra lúc soát:** núi = ô `Mountain` 2D thành tường cao 3–7 khối (`VoxelSolidSector` BFS `min(2+d,6)`); độ cao đắp sau (`VoxelHeightField` macro 40 + đồi 5 khối); bản build thật vẫn chạy `LocalMapGenerator` một ô 200² vì công tắc thế giới liền chỉ bật trong Editor (EditorPrefs).

**Quyết định user (không diễn giải lại):**
- Trục đứng KHÔNG giới hạn về kiến trúc ("không nên tự giới hạn kiến trúc sau này"), nhưng KHUNG SINH cố định Y 0..512 tính từ đá nền. Hiện `BlockLayers.Count = 256`, `Zero = 64`, `VoxelChunk.Sections = Count/16` cố định, ~70 chỗ dùng trong 15 file ⇒ phải đổi sang cột section thưa + tầng int.
- Sông: chảy thật từ mạng sông world (`RiverPass`) — rộng dần, mặt nước hạ theo dốc, ghềnh/thác, cắt núi thành hẻm (hơn MC).
- Hình địa hình: world (biển/núi/sông/khí hậu) + spline kiểu MC + hồ sơ địa hình từng biome bằng JSON kiểu Hytale, trộn liền qua mép.
- Mực biển = Y 160 (chốt). Mốc đề xuất: đá nền 0–4, lòng đất 5–150, đáy đại dương 100–140, đồng bằng 162–180, đồi ~220, núi 300–450 (đỉnh hiếm ~480), trên 512 không sinh nhưng xây được — tất cả JSON.
- Biển/viền thế giới: CHƯA chốt ("chưa có viền cụ thể hiện giờ") — giữ cả hai chế độ, đừng tự chọn.
- Config bằng JSON (toàn cục + từng biome), muốn có cửa sổ xem trước.

**Giai đoạn (làm qua đêm 2026-10-19 trên nhánh `worldgen-3d`, CHƯA gộp master — user sẽ test sáng rồi giữ/bỏ):**
- ✅ `0102c1ee` dọn đường gen cũ: thế giới liền là đường duy nhất; gỡ LocalMapGenerator + 5 pass + preview + prototype seamless.
- ✅ `87e7bf5b` 3D trong bản build: `VoxelGameView` tự gắn + tự bật trong build (Editor vẫn menu + F9); 2 shader voxel vào Always Included. Chưa chạy thử bản build thật.
- ✅ `bf53238b` trục đứng không giới hạn: lát thưa nối thêm theo nhu cầu (chunk + ánh sáng + bộ gộp mặt); `BlockLayers.GenTop`=512, `Limit`=2^20 (trần kỹ thuật); `Count` bỏ; `Zero` lúc đó GIỮ 64 (sau dời 160) (save lưu tầng tuyệt đối ở nhiều chỗ: khối sửa, đồ rơi `Layer`, công trình `BlockLayer`, designation z — dời Zero phải chuyển đổi hết). Test Vertical Axis 19.
- ✅ `a49bd046` hàm độ cao `World/Terrain/TerrainShape` (chưa nối vào game): Catmull-Rom 4×4 ô world + spline JSON + ridged xoay/uốn + hồ sơ `BiomeDefinition.terrain`; biển 160, trần mềm 420→490. Xem trước menu `Nizhangjie/WorldGen/Xem trước địa hình 3D`. Test Terrain Shape 13. ~305 ms/ô world trên Mono ⇒ cần Burst trước khi nối.
- ✅ (2026-10-19 chiều, user: "bỏ save thôi … cậu làm tiếp được mà") `86db865f` Zero 64→160 (save cũ lệch 96, user chấp nhận) + tăng tốc dựng chunk; `0cee5a4e` hàm độ cao Burst ≡ C# từng bit (22 ms/ô); `42d31b59` lát nén 4 bit (800→203 MB / 25 ô núi); `5b020214` NỐI VÀO GAME: `ContinuousMapBuilder` phân loại ô theo độ cao (biển/cát/vách theo dốc), `VoxelWorld` dựng khối từ `TerrainShape.Fill` qua `VoxelHeightField.BuildFromHeights`, đá không mọc vách. Play Mode: núi + bờ biển đẹp, 25/25 ô, console sạch. Ô núi dựng ~1,6 s (luồng nền, hang chiếm phần lớn).
- ✅ (2026-10-19 tối, user: "quá thực, núi quá dốc, đồng bằng quá phẳng, không giống MC/Hytale") hình 'ra game': núi thấp (đỉnh ~sea+150), đồng bằng lượn sóng (`rollBlocks`), bậc thềm THEO MẢNG trên núi (`terrace*`; bậc đều khắp núi = đường đồng mức, user chưa thấy, đã bỏ), xoay tầng đầu Ridged (hết vệt thẳng). Chờ user xem ảnh `Logs/TerrainPreview/gamey/`.
- ✅ (2026-10-19 khuya, user: "sông, hồ và biển thật, lấy từ mạng sông của world cái đi, rồi ta tinh chỉnh sau") SÔNG/HỒ THẬT: `TerrainShape.Hydrology.cs` (BFS hướng chảy từ biển, hồ = nút, nhánh chính, Chaikin + uốn khúc, mặt nước chỉ hạ ⇒ hẻm/ghềnh, mặt hồ = bờ thấp nhất − 1), lõi khắc Burst ≡ C#, `Fill(…, water, kind)` → ClassifyTiles + BuildFromHeights; pass 2D sông/hồ tắt. Test Terrain Shape 24. Chờ user tinh chỉnh (bề rộng/thung lũng/bờ hồ trong JSON).
- ✅ (2026-10-19, user: "cây phải cách ít nhất qua tán lá của nhau") cây giãn theo tán: `TreeShapes.MinTrunkSpacing` (tán A + tán B + 1 + canopyGap), sinh rừng = đĩa Poisson 4 pha theo toạ độ tuyệt đối trong `ContinuousFeatures.Generate` (lọc 1 lượt/Luby quá thưa — đã bỏ), mầm `TryGrow` cũng chặn chạm tán.
- ✅ (2026-10-20, user: "lúc rất phẳng để xây nhà, lúc lởm chởm lồi lõm, ngẫu nhiên") tính đất theo vùng `character*` trong terrain_shape.json: êm 20% / gồ 18% đất liền; noise PHẢI là Fbm + uốn (một tầng lộ lưới ô cờ).
- ✅ (2026-10-20) tool Editor `Nizhangjie/WorldGen/Bản đồ thế giới (tinh chỉnh địa hình)` = WorldMapTunerWindow: chỉnh terrain_shape.json trên bản đồ cả world, lớp xem, đặt ô xuất phát; dùng nó khi user muốn tinh chỉnh địa hình.
- ✅ (2026-10-20, user: "bỏ rìa là biển được không") islandFalloff=false cả 2 bản worldgen_config (2 bản file đó vốn lệch nhau). Vá kèm: ClassifyTiles lấy đất nền theo biome tại chỗ (trước theo cả ô ⇒ vết nối ở ô núi).
- ✅ (2026-10-20, user "okay thử đi" sau khi hỏi vì sao MC/Hytale méo mà thật) A uốn lớp chi tiết (detailWarp*) + B trục đỉnh–thung lũng (pvScale/peaksValleys). C ĐÃ LÀM (dưới); còn D hình địa hình riêng từng biome.
- ✅ (2026-10-20, "làm luôn đi") C điêu khắc 3D quanh mặt đất: heightmap giữ làm nền, WorldGen3D.SculptChunk khoét/mọc trong dải ± biên độ (sculpt* trong terrain_shape.json, mạnh ở gồ/núi, 0 ở êm và sát nước); phần mọc thêm phải liền khối dưới (không đất lơ lửng); tool có Lát cắt. ⚠ IBlockTerrain.TerrainLayers vẫn là trước điêu khắc.
- Đo 2026-10-20 (Play Mode, dựng lại 25 ô quanh (120,108), luồng nền): 4d9f33b0 (trước tính đất/PV/điêu khắc) TB 1049 ms/ô (955–1230) ⇒ 56a86e65 TB 1343 ms/ô (1174–1667), +28%. Cách đo: reflection VoxelWorld.Build(sv) rồi chờ _built tăng; mỗi execute_code ≤ 9 ô kẻo MCP timeout.
- Còn (sau sông/hồ): nội dung theo độ cao; nước sông chảy thật (hệ chất lỏng); sông/hồ/biển thật (sông vẫn từ pass 2D, mực = min quanh khúc; hồ world ngang mặt biển) → nội dung theo độ cao (đá lẫn chỉ ở tầng 106–168, tuyết, cây theo độ cao) → sương mù cho núi xa; hang có thể Burst nếu nạp chậm.
- (cũ) Kế: nối hàm độ cao vào thế giới khối (thay `VoxelHeightField` + núi-tường của `VoxelSolidSector`; loại ô suy từ độ cao/độ dốc; quyết Zero: nền mới ở tầng ~160+ trong khi Zero=64) → sông/hồ/biển đục vào hàm → nội dung theo độ cao. Núi cao tăng mặt vách: nhờ tài khoản kia đo FPS/RAM.
**Bẫy đã dính:** mặt hồ số nguyên qua nội suy Catmull-Rom thành 159,99998 ⇒ nước tụt 1 tầng (làm tròn lại); sông đổ vào hồ giữ mực cũ trong khi bờ hồ bị hạ ⇒ tường nước (hạ mặt sông theo nhịp bờ); Burst ⟂ Mono lệch tới 3 khối vì độ dời noise lớn (~41000) + trung gian double của Mono — sửa bằng mod 256 + làm tròn từng phép (§C9); test có tầng tuyệt đối ghi cứng (Multi-Floor bể chỗ đứng) vỡ khi dời Zero; gọi `Generate()` lúc Play ghi vị trí avatar dưới khoá ô mới ⇒ lần sau vào ô đó bị kéo về chỗ cũ (thử ô khác). test gọi hàm private qua reflection (`VoxelGreedyTests` → `BeginGreedy`) vỡ khi đổi chữ ký; Perlin ridged tạo nếp gấp thẳng chéo dài — xoay mỗi tầng; phép đối chứng "tắt X thì test đỏ" phải quét đúng vùng (bản đầu quét 1 vùng nên không đỏ).

Liên quan [[project_block_world]], [[project-seamless-prototype]], [[project-localmap-wildgen]], [[reference_fluid_research]].
