---
name: project-render-origin
description: Gốc vẽ lớp 3D + nội suy thân vật lý (2026-10-15) — hết rung camera do float ở toạ độ ~7·10⁴ và do bước vật lý 60 Hz không nội suy; quy tắc R()/D() khi thêm code vẽ mới; commit 884f16c
metadata:
  node_type: memory
  type: project
  originSessionId: d93f285a-5ead-48f8-9bcd-66fa9efac21c
  modified: 2026-10-15T15:18:37.642Z
---

User báo (2026-10-15): "đường kẻ nứt" ở vài góc nhìn + màn hình rung liên tục khi di chuyển.

**Rung — 2 nguyên nhân, đã sửa + đo:**
1. Toạ độ thế giới liền ~72 700 ⇒ float chính xác ~8 mm ⇒ camera/đỉnh nhảy bậc. Sửa: `VoxelGameView._origin` (bội cạnh ô world, chọn lúc TurnOn, cố định cả phiên). Logic giữ KHÔNG GIAN KHỐI tuyệt đối; chỗ GHI ra Unity dùng `R(v)` (trừ gốc), chỗ ĐỌC từ Unity dùng `D(w)` / `CamPos` / `ScreenRay` (cộng gốc). Camera quỹ đạo sống hẳn trong không gian vẽ (Follow/GroundHeight/PanBounds/Focus/Target đổi qua gốc). Lớp phủ `AddQuad` trừ gốc khi thêm đỉnh. `Drawn` có `P` (khối, bắt chuột) + `PR` (vẽ, nhãn).
2. Thân vật lý bước cố định 1/60 không nội suy ⇒ ở 120 FPS nửa số khung đứng yên. Sửa: `KinematicBody.InterpolatedRelativeTo(origin)` (nội suy giữa trạng thái trước/sau bước theo `_residual`, trừ gốc trên phần NEO nguyên ⇒ chính xác; đặt `Position` ⇒ bỏ nội suy lần đó), `WildPlayerToken.BodyFeetRelativeTo(origin)`; avatar (hình, camera POV + quỹ đạo, bong bóng) đọc qua đây.
Test `Run Kinematic Body Tests` 21 (+4 Interp_*): cách cũ đứng yên 40/80 khung + lệch bước 6%, cách mới 0/80 + 0%.

**"Đường nứt" — CHƯA tái hiện được:** đếm điểm nền (camera nền magenta) dưới chân trời ở 72 góc: 0 lỗ cả cách cũ lẫn mới ⇒ không phải khe hình học chữ T ở các góc thử. Điểm magenta lẻ chỉ ở đường chân trời (trời lọt kẽ lá xa, hợp lệ). Ứng viên: viền ô đang trỏ (`_hover`, nét mảnh vàng nhạt, nhìn xiên = chấm chấm) hoặc nhãn tên ở xa — chờ user xác nhận sau bản sửa.

**Bẫy đo:** đặt camera bằng tay rồi `cam.Render()` trong cùng lệnh ⇒ nhãn TMP vẫn xoay theo camera của LateUpdate trước ⇒ nhìn như vệt chéo (đã nhầm là lỗi). Đổi POV xong phải đợi 1 khung mới chụp. Vá kèm: `VoxelPovHud.UpdateHotbar` NRE khi Update chạy trước khi HUD dựng xong.

**Quy tắc cho code vẽ mới ở lớp 3D:** mọi vị trí đưa vào Transform/mesh của Unity phải qua `R()`; mọi vị trí/tia đọc từ camera/Transform phải qua `D()`/`CamPos`/`ScreenRay`. Quên ⇒ vật hiện cách đúng chỗ ~72 600 khối (vô hình) hoặc bắt chuột lệch.

Liên quan: [[project_kinematic_body]], [[project_shader_light_greedy]], [[project_voxel_view_prototype]].
