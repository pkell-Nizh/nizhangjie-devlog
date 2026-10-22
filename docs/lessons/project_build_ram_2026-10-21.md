---
name: project-build-ram-2026-10-21
description: "RAM bản build Development (Windows, Mono) 2026-10-21 — màn hình chính ~0,6 GB, trong thế giới 3D ~3,3 GB working set / ~7,1 GB private; Editor 5,7 GB là do Editor cộng dồn"
metadata:
  node_type: memory
  type: project
  originSessionId: e0aa2828-7179-4c4f-ad0c-14e042210922
  modified: 2026-10-20T19:19:04.551Z
---

Đo 2026-10-21 trên nhánh `worldgen-3d` (sau e1ae07cb). Build Development + Autoconnect Profiler, ra `C:/Users/buihu/NizhangjieBuild/` (ngoài repo). Lấy mẫu Get-Process mỗi 5 s, user tạo thế giới mới từ MainMenu rồi đi bộ ~2 phút.

- Màn hình chính: working set ~0,6 GB, private ~0,95 GB.
- Vào thế giới (khoảng 20 s nạp): nhảy lên 2,6 GB WS / 5–6 GB private.
- Sau 2 phút đi lại: 3,3 GB WS / 7,2 GB private, vẫn còn nhích lên chậm (chưa phẳng hẳn).
- Private 7 GB > WS 3,3 GB: một phần là bộ nhớ đã commit nhưng không chạm (driver D3D12, pool NativeArray cấp trước). Máy 8 GB RAM sẽ chật. Chưa phân rã theo nhóm — bước tiếp theo là nối Memory Profiler vào build.
- Editor từng hiện 5,7 GB: phần lớn là Editor cộng dồn. Log Editor lúc tắt báo nhãn `FontEngine` giữ 1,4 GB (không code nào tự tạo font; file Fallback chỉ ~1 MB) ⇒ nghi Editor giữ qua nhiều lần reload, chưa chứng minh.
- User báo (2026-10-21): treo game 5 giờ thật (~1 tháng 18 ngày trong game) RAM gần như không tăng ⇒ không rò rỉ theo thời gian mô phỏng. Chưa thử: đi xa liên tục qua nhiều ô (rò rỉ khi nạp/gỡ ô). Task Manager của user: ~2,8 GB, CPU ~30%, GPU ~35%.
- Bẫy khi build qua MCP: `manage_build` làm mất kết nối MCP và lệnh build biến mất. Cách chắc chắn là gọi `BuildPipeline.BuildPlayer` trong `EditorApplication.delayCall` qua execute_code, ghi kết quả ra file rồi chờ file.

Việc tối ưu RAM thuộc tài khoản perf, xem [[feedback-account-role-split]]. Liên quan [[project-load-order-view-circle]] (~28 MB/ô, 1,55 GB cố định đo trước).
