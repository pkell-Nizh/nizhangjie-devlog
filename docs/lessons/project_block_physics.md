---
name: project_block_physics
description: "Bước 5 lộ trình thế giới khối (2026-10-13) — cát/sỏi rơi kiểu Minecraft (FallingBlocks + KinematicBody), khối rơi trúng người trừ máu, đồ rơi đất có tầng (WorldItemDrop.Layer + GroundItemOps); bẫy vòng đời khối đang rơi"
metadata:
  node_type: memory
  type: project
  originSessionId: 73d00b3f-d9f4-41fc-89e1-cb95d0d9f854
  modified: 2026-10-13T13:49:14.909Z
---

ĐÃ CODE + TEST XANH (`Run Block Physics Tests` 36) + ĐÃ VERIFY Play Mode qua execute_code (chưa bằng phím thật), 2026-10-13. User chốt "theo đề xuất": chỉ cát + sỏi có trọng lực, khối rơi trúng người CÓ sát thương, đồ rơi KHÔNG hết hạn, KHÔNG kết cấu chịu lực.

- `BlockDefinition.gravity` (data: `block_sand`, `block_gravel`, 2 thư mục) → `BlockIds.Gravity`.
- `World/Voxel/FallingBlocks` (thuộc `BlockTicks.Falling`): khối RỜI SỔ thành vật thể `KinematicBody`, chạm đất `Write` lại; ô chạm đã có thứ khác hoặc đáp lên thứ không phải khối (mái, đồ đạc) ⇒ vỡ thành vật phẩm. Kích hoạt CHỈ ở `BlockTicks.OnBlockReplaced` (không quét map ⇒ cát lơ lửng worldgen chỉ sập khi bị chạm). Cả cột rơi nhờ đệ quy: khối rời đi báo khối trên.
- Sát thương: `IFallTargets` → `World/Wild/Combat/TokenFallTargets` (chỉ pawn: All + Visitors). Số ở `Data/voxel/block_ticks.json`: bỏ 1 khối đầu, 5% máu tối đa/khối, trần 40%.
- Đồ rơi: `WorldItemDrop.Layer` = tầng tuyệt đối của ô nó nằm, **0 = sàn chính** (mọi đống kiểu cũ/save cũ); `Cell` vẫn là cột z=0. `World/Wild/Storage/GroundItemOps`: `Drop` (gộp chồng cùng loại + cùng chủ + cùng tầng), `OnBlockChanged` (mất đỡ rơi tiếp / bị đè đẩy lên), `SpotOf`, `Approach` (nhặt trong hang qua `BlockWork.FindApproach`). Quãng rơi của ĐỒ chỉ là hình (`VisualHeight`), tầng nghỉ chốt ngay.
- `BlockPlacementOps.Give(…, dropAt)`: `dropAt.z` = tầng thả; job truyền `token.Spot`.

**Bẫy:**
- Khối đang rơi KHÔNG nằm trong `BlockStore` ⇒ phải `SettleAll` trước khi chụp save / đổi map / tắt VoxelWorld (đã nối ở `SaveManager.WriteFile`, `VoxelWorld.Relayout/OnDisable`, `SettleIn` khi ô world rời cửa sổ). Thêm đường lưu mới mà bỏ qua `WriteFile` là mất khối.
- Đặt khối lên chỗ "dưới không phải khối đặc" sẽ rơi lại vô hạn ⇒ `Land` yêu cầu ô ngay dưới là KHỐI đặc, không thì vỡ.
- Heredoc bash chứa `'''` hỏng ở tool Bash ⇒ ghi script Python bằng tool Write vào scratchpad rồi chạy (helper `ed.py`: `sub()` tự xử CRLF — file .cs cũ là CRLF).

**Nợ:** quái/linh thú/dân chưa bị khối rơi trúng; khối vỡ trên mái thì vật phẩm lọt xuống sàn nhà; kéo-thả / túi POV vẫn thả đồ ở sàn chính; cột cát cao rơi tốn ~3,4 ms × 2 mỗi khối; CHƯA commit.

Liên quan: [[project_world_like_minecraft_roadmap]], [[project_block_world]], [[project_kinematic_body]], [[project_feature_to_blocks]], [[project_multifloor_pathfinding]].

**Sửa kèm (2026-10-13, lỗi vẽ dưới tán cây):** `VoxelChunkSet.SkyTop` trước tính cả lá/thân cây là "đất đá che" ⇒ mặt đất + thân + mặt dưới tán bị xếp vào lưới HANG (chỉ dựng trong 56 khối quanh camera) ⇒ nhìn xa dưới mọi tán là lỗ thủng. Nay `RecomputeColumn` bỏ qua `TreeShapes.IsLeaf/IsLog` khi tính trời. Giá: tổng đỉnh lớp 3D ở WildLocal 9 ô world ~8,36 triệu (chưa đo số trước khi sửa trong cùng điều kiện). Nếu cần giảm: gộp mặt (greedy) hoặc LOD tán cây, ĐỪNG đưa tán về lại lưới hang.

**Số đo sửa tán cây (2026-10-13, WildLocal 9 ô world, lớp 3D bật, camera quỹ đạo mặc định, 1 ngày game ở 8x ≈ 150 s, mỗi bản 1 lượt):** luật cũ 124,7 FPS, khung chính avg 7,94 / p95 12,31 / p99 16,73 ms, 31 khung >33 ms, tổng đỉnh 4,72 triệu ⟂ bản sửa 110,1 FPS, avg 9,01 / p95 13,35 / p99 17,86 ms, 30 khung >33 ms, tổng đỉnh 8,49 triệu; đỉnh vẽ mỗi khung 557k → 596k. Bẫy đo: bắt đầu ghi ngay sau `TurnOn` thì dính lúc lớp 3D còn dựng hình (lượt hỏng: p95 40 ms, 821 khung >33 ms) ⇒ chờ >1000 khung rồi mới ghi. Đo bằng `ProfilerRecorder` ("Main Thread", "PlayerLoop", "Vertices Count") giữ qua `AppDomain.SetData`.
