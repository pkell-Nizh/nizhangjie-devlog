---
name: project_block_light
description: "Ánh sáng theo khối (bước 6 lộ trình MC, 2026-10-14) — 3 quyết định của user, cả 5 phần xong (loang sáng, vẽ ánh sáng mịn, khối đuốc, sinh vật tối theo chỗ đứng, loài khai thì chỉ sinh chỗ tối); phần 1–4 đã thấy trong Play Mode, phần 5 mới có test"
metadata:
  type: project
---

**User chốt 2026-10-14 (đừng hỏi lại):** hang TỐI ĐEN như Minecraft (không có sáng nền); ánh sáng MỊN kèm bóng góc ngay từ bản đầu; đuốc cầm tay tự sáng để SAU khi vẽ ánh sáng đã tốt. User cũng bảo học pipeline của modder Minecraft trước — xem [[reference_minecraft_mod_pipelines]].

**Phần 1–5 XONG (test `Run Block Light Tests` 41 xanh; Play Mode 2026-10-14 đã chụp ảnh `Assets/Screenshots/light_*.png`; CHƯA commit):**
- `World/Voxel/VoxelLightField` — hai kênh 0..15 (trời ở 4 bit cao, khối ở 4 bit thấp), mảng byte theo lát 16³ song song `VoxelChunk`, lát đồng nhất không cấp mảng. `ComputeAll()` thuần (chạy trong `VoxelChunkSet.Build`, luồng nền, chỉ trong hình đó); `OnBlockChanged` gỡ hai lượt rồi loang lại (gọi từ `VoxelChunkSet.Write` khi `recompute`). Đọc: `set.Light.Sky/Block/Packed(x, tầng, y)` — toạ độ CỤC BỘ của hình, đọc xuyên sang hình kề đã nối.
- Hình kề: `VoxelLightField.Link` + `ExchangeBorders` + `Unlink`, do `VoxelWorld.LinkLight` gọi trong `Apply` (và `Unlink` khi gỡ / thay bản chunk). Sau khi nối, loang / gỡ / đọc đều đi xuyên ranh. Hàng đợi loang là `[ThreadStatic]` (luồng nền tính cả hình ⟂ luồng chính sửa khối).
- `BlockDefinition.lightEmit` (0..15) + `lightOpacity` (-1 = theo `solid`: đặc 15, không đặc 0); `BlockIds.Emit/Opacity/AnyEmitter`. Data: mọi khối lá `lightOpacity: 1`, `block_lava` `lightEmit: 15`.
- Vẽ: `VoxelMapMesher.LitFace` — mỗi đỉnh lấy trung bình tối đa 4 ô không đặc quanh nó (ánh sáng mịn) + bóng góc nhân vào màu đỉnh; cắt tam giác theo đường chéo hợp. Ánh sáng nằm ở TEXCOORD0 dạng 4 byte (x = 1 − trời, y = khối) ⇒ đỉnh 20 byte (3 luồng đỉnh, `Acc.ToMesh`). Lưới KHÔNG có kênh này (hình nộm, hộp rời, mái, lớp phủ) đọc ra trời đủ ⇒ y như cũ. Shader `VoxelView.shader`: `max(curve(trời) × _Light, curve(khối) × _BlockLight)`, sàn `_MinLight` 0,02, `curve(l) = l / (4 − 3l)`.
- Dựng lại lưới: chunk có KHỐI đổi dựng ngay như cũ; chunk chỉ đổi ÁNH SÁNG đi qua `VoxelWorld.ChunkLightChanged` → `VoxelGameView.UpdateLightRebuilds` (3 chunk mỗi khung, chờ nếu hình đang phát mặt ở luồng nền).
- Đo Play Mode (9 ô world có thành phố): ~300 lát ánh sáng cấp mảng mỗi ô world (~1,2 MB); dựng lại một chunk 1,5 ms; tổng 8,36 triệu đỉnh (không tăng); phát mặt một ô world ở luồng nền 937 ms (CHƯA có số trước khi thêm ánh sáng để so).
- Không lưu save (suy ra từ khối).

- Đuốc (phần 3): khối `block_torch` (không đặc, `needsSupport`, `lightEmit: 14`) ↔ vật phẩm `torch`; công thức `recipe_torch` = 1 `wood_plank` + 1 `coal` → 4 đuốc, làm ở bàn mộc hoặc bếp lửa (game CHƯA có chế tác tay không). Hình vẽ: khối không đặc mà phát sáng thì `VoxelMapMesher.EmitPlant` vẽ cán gỗ + đầu lửa tự sáng. Chưa có đuốc gắn tường, chưa có đuốc cầm tay tự sáng (user bảo để sau).
- Sinh vật + đồ rơi (phần 4): `VoxelWorld.LightAt(điểm)` trả byte ánh sáng tại một điểm trong không gian khối. Hình nộm đơn vị: `VoxelGameView.SetLook` đặt `_UnitLit` từng renderer (shader cộng vào kênh lit). Đồ rơi / khối đang rơi vẽ bằng lớp phủ: `LitOverlay` quy đổi độ sáng trên CPU. Màu ánh đuốc + sàn sáng khai MỘT chỗ ở `VoxelGameView.BlockLightColor/MinLight` (đặt vào vật liệu lúc tạo).

- Quái sinh chỗ tối (phần 5) — user chốt 2026-10-14: "TUỲ LOẠI quái thì mới sinh trong bóng tối, sau này sẽ có bản sinh sinh vật mới tốt hơn" ⇒ làm NHỎ, theo loài, đừng đầu tư thêm vào bộ sinh hiện tại. `BeastSpeciesDefinition.spawnMaxLight` (-1 = không xét; 0..15 = chỉ sinh khi ánh sáng tại ô ≤ mức đó). `BeastSpawnOps.EffectiveLight` = max(ánh sáng khối, trời − `NightPenalty(giờ)`), đêm trừ 11 ⇒ loài khai 7 sinh ngoài trời ban đêm + chỗ kín ban ngày, đuốc chặn được. `BeastSpawner.TrySpawnOnePack` lọc theo ô (chỉ con đầu đàn; con theo đàn không xét). Loài mẫu duy nhất đã khai: `species_bao_forest` (U minh báo) = 7 — em tự chọn làm ví dụ sống, user chưa duyệt loài nào. CHƯA thử Play Mode (sinh ngẫu nhiên). Quái vẫn chỉ đứng sàn chính ⇒ chưa sinh được trong hang ngầm.

**Play Mode 2026-10-14 (lần 3):** đuốc đặt trong HANG TỰ NHIÊN (tầng 50, sâu 21 khối) chiếu sáng vách hang đúng (`Assets/Screenshots/light_cave_torch_*.png`). Lỗ xanh thấy trong ảnh b là mép cửa sổ thế giới đã nạp (ngoài hình không có mặt nào) — có từ trước, không phải lỗi ánh sáng.

**CHỜ USER QUYẾT (đã hỏi 2026-10-14):** mái nhà thành phố có chắn sáng không. Mái không phải khối và góc nhìn toàn tri ẨN mái ⇒ nếu cho mái chắn sáng thì nhìn từ trên xuống trong nhà sẽ tối om. Đừng tự làm.

**Nợ / bẫy:**
- Hình bị dựng lại hoặc gỡ: ánh sáng nó ĐÃ tràn sang hình kề để nguyên (không gỡ) — nguồn sáng sát ranh bị mất theo cách đó sẽ để lại vệt sáng tới khi hình kề dựng lại. Hiếm; chưa xử.
- Mái nhà (`data.Roofs`) không phải khối ⇒ không chắn sáng; trong nhà có mái vẫn sáng như ngoài trời.
- Nước / dung nham hiện là khối ĐẶC ⇒ chắn sáng hẳn — xét lại ở bước 7 chất lỏng.
- Hộp rời / đồ đạc (props), nhãn tên, bóng vật bay, hình avatar ở POV (`VoxelPovHud`) CHƯA tối theo ánh sáng.
- ĐỪNG gọi `VoxelMapMesher.RebuildChunk` tay khi thử trong Play Mode mà không gán lại `MeshFilter` — nó huỷ lưới cũ, GameObject trỏ vào lưới chết (đi qua `OnChunkChanged` mới đúng).
- Bản cập nhật dần phải LUÔN bằng bản tính lại từ đầu — test `Incremental_EqualsFullRecompute` so từng ô; thêm luật loang mới thì giữ test này.
- GPU vẫn nghẽn ở số đỉnh; ánh sáng mịn làm mặt kề nhau khác màu ⇒ gộp mặt sau này gộp được ít hơn.

Liên quan: [[project_world_like_minecraft_roadmap]], [[project_block_world]], [[project_fps_profile_2026-10-14]].
