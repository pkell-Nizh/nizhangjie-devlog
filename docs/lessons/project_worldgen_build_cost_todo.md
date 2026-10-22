---
name: project-worldgen-build-cost-todo
description: VIỆC CHO TÀI KHOẢN PERF — dựng ô world trên worldgen-3d chậm hơn 28% sau tính đất/PV/điêu khắc 3D (1049 → 1343 ms/ô); cần tối ưu nếu được
metadata:
  node_type: memory
  type: project
  originSessionId: 62ee1e22-0d21-42dc-b346-287ab08d6fa3
  modified: 2026-10-19T21:17:01.687Z
---

**Nhờ tài khoản perf xem** (user dặn 2026-10-20, tài khoản còn lại chỉ làm nội dung).

Đo 2026-10-20 trên nhánh `worldgen-3d`: Play Mode, dựng lại 25 ô world quanh (120,108) bằng reflection `VoxelWorld.Build(sv)` rồi chờ `_built` tăng. Mỗi `execute_code` dựng tối đa 9 ô, nhiều hơn thì MCP timeout.
- `4d9f33b0` (trước session 2026-10-20): trung bình **1049 ms/ô** (trung vị 1042, 955–1230).
- `56a86e65` (thêm tính đất êm/gồ, uốn toạ độ chi tiết, trục đỉnh–thung, điêu khắc 3D): trung bình **1343 ms/ô** (trung vị 1325, 1174–1667).
- Chậm hơn **+28%** (từ +11% tới +75%, ô (118,110) 955 → 1667 ms).
- Việc dựng chạy ở luồng nền nên không giật khung, nhưng ô mới hiện chậm hơn khoảng 0,3 s.

**Why:** user muốn giữ hình địa hình mới nhưng dựng nhanh lại nếu được. Phân vai: perf thuộc tài khoản kia ([[feedback-account-role-split]]).

**How to apply:** đo phân rã trước khi sửa ([[feedback-measure-before-optimizing]]). Mới biết điêu khắc 3D tốn khoảng 150–200 ms/ô; phần còn lại (khoảng 100 ms) CHƯA đo riêng. Chỗ nghi ngờ:
- `WorldGen3D.SculptChunk` + `SculptNoise`: `Fbm3` trên lưới nút 4 khối, nội suy tam tuyến, C# thường chưa Burst.
- `TerrainShape.SculptStrength` (lưới 8 khối + bilinear) và `VoxelWorld.SculptAmplitude` (khoảng cách Manhattan tới nước).
- `TerrainShapeCore.Sample` thêm Fbm cho tính đất, PV, uốn toạ độ (2 N11). Có Burst nhưng nhiều lần lấy mẫu hơn.
- `ClassifyTiles` theo biome từng ô (`_groundOfBiome`).

Ràng buộc khi tối ưu:
- `TerrainShapeCore.Sample` phải giữ Burst ≡ C# từng bit (test Seamless Burst).
- Điêu khắc phải tất định theo toạ độ tuyệt đối (test `Sculpt_*` trong Terrain Shape 31).

Xem [[project-worldgen-3d-redesign]].
