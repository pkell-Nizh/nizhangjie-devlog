---
name: project-voxel-z-future-proofing
description: "Bàn về nâng cell system lên voxel (z=0 ground/z=1 non-ground) để chuẩn bị game 3D ~2028-2030 — đã khảo sát, CHƯA implement, chờ user nhờ lúc rảnh"
metadata: 
  node_type: memory
  type: project
  originSessionId: 4d325e7a-f541-4f2b-9ba8-6c8152d8a4d3
  modified: 2026-10-08T09:44:37.087Z
---

**2026-09-11, user đặt vấn đề:** có nên "nhân dày" hệ cell 2D hiện tại (`Cell[]` phẳng, xem [[project-cellindices]])
thành voxel không, để phòng khi dự án lớn thành game 3D thật vào 2028-2030, tránh modder vỡ API và tránh
bản thân user phải rewrite trong tuyệt vọng. Ý cụ thể: chỉ cần 2 tầng Z (0=ground, 1=non-ground), không
phải voxel liên tục.

**Đã khảo sát bằng code (không đoán):**
- KHÔNG có 1 struct tọa độ tự viết để chêm field `z` vào — toàn bộ **251 file / ~1961 chỗ dùng thẳng
  `UnityEngine.Vector2Int`** (type built-in Unity, chỉ x,y). Muốn thêm z ở tầng ENGINE sau này = đổi
  `Vector2Int`→`Vector3Int` (Unity có sẵn) toàn cục — cơ học, compiler dẫn đường (constructor 2 tham số
  đỏ ngay), nhưng là refactor LỚN thật (251 file), không nhỏ như ước tính ban đầu.
- **Tin tốt — save/JSON đã AN TOÀN sẵn:** `SaveManager` dùng `JsonUtility`; mọi snapshot struct
  (`ZoneTileEntry`, `GroundItemSnapshot`, `PawnPosition`, `WildMapBuildState.*`) đều tách phẳng
  `int x, int y` tường minh, **KHÔNG BAO GIỜ serialize `Vector2Int` trực tiếp** — quy ước đã cố ý áp dụng
  từ trước (comment tự giải thích ở [WildMapBuildState.cs:158](Nizhangjie/Assets/Scripts/World/Wild/Persistence/WildMapBuildState.cs:158):
  "JsonUtility không serialize Vector2Int trong HashSet gọn"). Nghĩa là thêm `int z = 0` vào các struct
  save này bây giờ HAY để dành đều non-breaking (JsonUtility để field vắng mặt trong JSON cũ về default
  C# = 0 = ground).

**⚠ CẬP NHẬT 2026-10-08 (đã kiểm git + grep): migration ĐÃ LÀM ngày 2026-09-12** — commit `b8010d4`
(baseline) → `92f3188` (Grid/Zoning/Pathfinding/Combat/World/Wild) → `f3e98fc` (Tests/Editor, toàn project 0
lỗi). Nhật ký: `Assets/Scripts/VECTOR3INT_MIGRATION_NOTES.md`. Hiện 242 file dùng `Vector3Int`, 97 file còn
`Vector2Int` — CỐ Ý chưa đổi: `World/City` (36), `Sim/Npc`, `Hamlet`, `LocalMap/Buildings/Building.cs`,
`Core/Pawns/Capture`, `SampleEvents`; nối bằng ép kiểu ở biên. z LUÔN = 0, chưa tính năng nào dùng z=1.
Tầng nhà City (`MultiFloorPlanner` floor0/floor1) là hệ RIÊNG, không đi qua z. Phần "hoãn" bên dưới là lịch sử.

**Kết luận đã chốt cùng user (lịch sử, 2026-09-11):**
- Phần schema save/mod (đổi thật, không sửa muộn được) — AN TOÀN SẴN, không cần làm gì thêm ngay.
- Phần engine (`Vector2Int`→`Vector3Int` 251 file) — hoãn, KHÔNG làm bây giờ (chưa có tính năng nào cần
  z=1, làm bây giờ là thiết kế cho tương lai giả định, xem [[feedback_core_sim_before_xianxia]]/nguyên
  tắc YAGNI dự án đang theo). **User chốt: "tới lúc đi ngủ sẽ nhờ sửa 251 file"** — tức là việc này ĐANG
  ĐƯỢC HOÃN CÓ CHỦ Ý tới khi user chủ động yêu cầu, không phải quên. Khi được nhờ: đây là refactor cơ học
  lớn, nên làm cẩn thận (biên dịch từng bước, không đổi hành vi runtime — mọi Vector3Int mới mặc định
  z=0).

**Liên quan:** [[project-cellindices]] (Cell struct AoS, KHÔNG liên quan Vector2Int — Cell là world/local
grid data, Vector2Int là tọa độ runtime dùng khắp UI/AI/combat), [[feedback_measure_before_optimizing]].
