---
name: project_hitch_profile_2026-10-15
description: "Đo GIẬT CỤC (không phải FPS trung bình) ở WildLocal 5×5 ô world (2026-10-15) — đập khối, qua ranh ô world, giật định kỳ; thủ phạm + số ms; chưa sửa gì"
metadata:
  node_type: memory
  type: project
  originSessionId: d93f285a-5ead-48f8-9bcd-66fa9efac21c
  modified: 2026-10-15T08:52:47.394Z
---

Đo 2026-10-15 trong Editor (Profiler bật, cửa sổ 1100×500, `streamRadius` 2, thành phố), sau khi [[project-fps-profile-2026-10-14]] cho thấy FPS trung bình đã ổn (~124). CHƯA sửa gì — chờ user chọn.

**1. Đập 1 khối:** `BlockPlacementOps.Break` đứng 5–13 ms liền. Gần hết là `VoxelWorld.OnBlockChanged` → `ChunkChanged` → `VoxelGameView.OnChunkChanged` dựng lại lưới chunk NGAY TRONG sự kiện (chunk thường ~2 ms, chunk thành phố có lưới hang gần camera ~6 ms). Một khối thì chịu được; chuỗi đổi ô (nước chảy, cát rơi, lá rụng) = mỗi ô dựng lại chunk thêm 1 lần ⇒ nên gom "chunk bẩn" rồi dựng 1 lần/khung (kiểu MC). Các listener khác rẻ (WildTilemapRenderer ~0,3 ms, PathGrid 0,05).

**2. Qua ranh ô world (nạp 5 ô mới):** tổng ~1 s khựng. Lúc ghép từng ô: `LocalMapBootstrap.Update` 9–56 ms + `VoxelWorld.Update` 9–20 ms mỗi lượt. Ngay sau khi ghép xong 4 khung 180–340 ms: `BeastBehaviour.Update` 65–118 ms, `CityNpc.DrainPathQueue` 42–117 ms (đường NPC của thành mới xử trên luồng chính), `VoxelWorld.Update` 24–90, `VoxelGameView.Update` 20–46. Suốt lúc đó `WildTilemapRenderer.ClearQueueRoutine`+`PaintQueueRoutine` ăn ~6 ms MỌI khung — lớp 2D vẫn tô tilemap dù đang xem 3D (2D đã đình trệ).

**3. Giật định kỳ khi đứng yên:** `DoorTrafficSystem.Update` 10–25 ms và `BeastBehaviour.Update` 8–23 ms, vài giây một lần ⇒ đây là thứ người chơi thấy thường xuyên nhất.

**4. `JobGiver_HaulToStorage`:** đồ rơi pawn không dùng được (trong nhà kín) ⇒ mỗi pawn mỗi nhịp chậm quét cả 686 kho của thành rồi thất bại (~3–6 ms/pawn ⇒ `PawnAiBrain.HandleTick` 23–43 ms). Lần đo do chính đồ rơi từ phép thử đập ngói mái gây ra, nhưng lỗ hổng thật: thiếu "nhớ thất bại" + quét mọi kho.

**Bẫy đo:** mỗi lệnh `execute_code` MCP tạo 1 khung ~1,2 s (`UnitySynchronizationContext.ExecuteTasks`) — loại ra khi đọc. Tăng `GameClock.SetSpeed(3)` làm giật nhịp chậm dày gấp 3. Bộ ghi khung dùng `Application.onBeforeRender` (khung giãn ghi ở lần gọi SAU). Bọc handler sự kiện bằng reflection (thay backing field delegate) để bấm giờ từng listener.
