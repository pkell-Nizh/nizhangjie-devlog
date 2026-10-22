---
name: project-dots-hybrid
description: "DOTS hybrid (2026-10-01): 7 luật; bước 2 (A* NPC) + bước 1 (worldgen seamless, noise tự viết) XONG; tham số worldgen chưa chỉnh, chờ user; Entities chưa cài. CLAUDE.md §6.1."
metadata:
  node_type: memory
  type: project
  originSessionId: 11e97d05-e2ad-42fa-bbed-77c43cb8a7b9
  modified: 2026-10-01T13:16:44.115Z
---

User chốt (2026-10-01): dùng Unity DOTS theo kiểu **hybrid**. Sơ đồ + 7 luật đầy đủ nằm ở `Assets/Scripts/CLAUDE.md` §6.1, đọc trước khi viết code Jobs/Burst/Entities.

Luật cũ "sim single-thread" có từ hồi sợ race condition. User cho rằng Safety System của Jobs đã xử lý chuyện này. M28 §8.1 "bác bỏ ECS-lite" là ghi chú lỗi thời, lỗi A* nói ở đó đã sửa từ lâu. Cả hai chỗ đã được đánh dấu trong doc.

Tóm tắt 7 luật: (1) sở hữu theo field: nóng thuộc DOTS, lạnh thuộc Registry; (2) điểm đồng bộ = `TickPhase` (Complete trước Ui, save sau đó); (3) sự kiện từ job đi qua hàng đợi, hook mod chỉ phản ứng; (4) hành vi trong hộp đen dựa trên data, hành vi mod tự định nghĩa đi đường chậm C#; (5) ID int chỉ trong RAM, dựng sau `GameDatabase.Freeze()`, save lưu `namespace:id`; (6) mỗi hệ DOTS giữ bản C# thường để test so sánh; (7) gom về một mô hình luồng là Jobs.

Lộ trình: (1) Burst cho `TileAt` + hashFeatures của seamless worldgen → (2) A* NPC thành phố sang Jobs → (3) tách `NpcRecord` thành `NpcHot[]` + record lạnh, chỉ khi Profiler cho thấy cần. **Entities chưa cài, không cài trước bước 3.**

Trạng thái 2026-10-01 (chưa commit, user dặn đừng commit vội):
- Burst/Collections/Mathematics đã ghim vào `manifest.json`.
- **Bước 2 XONG + verify Play Mode** — `LocalMap/Pathfinding/CellAStarNative.cs`, nối vào `CityNpcSystem` (`_useBurstPathJobs`). Ra đường giống hệt bản C# từng ô; nhanh ×5–7 trên thành phố 400×400. Test `Run CellAStar Burst Tests` 13/13.
- **Bước 1 XONG (cùng ngày, chưa commit)** — noise tự viết `PortableNoise` + lõi `TerrainCore`, `ContinuousTerrain.FillRect` gọi Burst trực tiếp (chạy được trên Thread của ChunkStreamer). Thế giới seamless ĐÃ ĐỔI; user dặn **chưa chỉnh tham số, cho xem so sánh trước** (menu `Nizhangjie/Seamless/So sánh noise cũ ⟂ mới`). 1 ca test `Wild_BorderBetterThanOldGenerator` đỏ do mẫu nhỏ, chờ user. Chi tiết CLAUDE.md §6.1.
- Mép thế giới seamless = chặn không đi qua (user chọn). Biển viền là `islandFalloff` có sẵn (2 chế độ).
- Bẫy đã dính (thành luật `CODE_SYSTEM_RULE.md` §C9): Mono không làm tròn float trung gian (phải ép `(float)` từng bước ở cả hai bản); `SortedSet` bỏ khoá trùng còn heap thì không; Burst Editor biên dịch bất đồng bộ nên test ngay sau khi sửa có thể chạy mã cũ.
- Test dự án chạy qua menu `Nizhangjie/Run … Tests`, KHÔNG qua Unity Test Runner.

- **Đo 2026-10-02 (8x Play Mode): chưa hệ nào cần thêm DOTS.** Wild chậm vì quét scene (B8), đã sửa ×14 (~5→0,36 ms/tick). City 1000 NPC: SlowTick 2,1 ms, frame p95 9,4 ms ⇒ bước 3 chưa đáng làm. Chi tiết + cách đo lại: CLAUDE.md §6.1.

**Why:** Entities không render sprite/tilemap 2D; mod nạp lúc chạy không bake được; API Entities hay đổi sẽ phá mod C#. Xem [[project-modding-strategy]].

**How to apply:** Mod không bao giờ chạm kiểu ECS/NativeArray. Đo Profiler trước ([[feedback-measure-before-optimizing]]). Determinism vẫn là luật.
