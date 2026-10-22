---
name: project-canreach-findpath-gap
description: "Vì sao CanReach true mà FindPath null ở Wild (2026-10-17) — region không nói dối; A* hành lang 3D chạm trần + Occupant công trình. Commit 63e34077 (nhánh claude/loving-brattain-c834ec, chưa merge main)"
metadata:
  node_type: memory
  type: project
  originSessionId: be8abb7f-28e5-428b-afd6-5d16062f1234
  modified: 2026-10-17T10:23:20.846Z
---

Lỗi "CanReach true, FindPath null" (đo WildLocal, 400 cặp quanh điểm vào (87240,55320), seed 12345): 87/252 cặp tới được bị lệch.
**Region KHÔNG nói dối** — PathRegionGrid flood/link dùng đúng `CanStepSpots` như A*.

Nguyên nhân thật:
- 80 cặp: đường thật rất vòng (vd thẳng 42 ô, đường 2431 bước quanh đồi), A* hành lang 3D (`SpotAStar`, cap 30000) chạm trần. Mục cũ (stale) trong hàng đợi vẫn được "mở" và ăn trần.
- 7 cặp (+4 sau sửa lộ ra): ô có `Cell.Occupant` là CÔNG TRÌNH (house_family, corner_store, clinic, battle_hall…). A* coi ô có Occupant là chặn (trừ đích), region bỏ qua occupancy ⇒ lệch. Bỏ qua occupancy thì 11/11 có đường. Chưa sửa — câu hỏi thiết kế: Occupant công trình nên là ô chặn trong PathGrid/region hay không.

Sửa (commit 63e34077, chỉ nhánh A* hành lang có tầng phụ): heuristic = max(octile, khoảng cách đồ thị region tới region đích); bỏ qua nút đã đóng; phá hoà f theo h; trần = max(30000, 2×số chỗ đứng hành lang). Đường octile/Burst giữ thứ tự cũ. Sau sửa: lệch 87 → 11 (toàn Occupant). Test `LongDetour_*` trong Run RegionPathfinder Tests (đỏ trên code cũ).

**Bẫy đo đã dính:** đo khi map chưa nạp xong (Regions.IsPendingAt còn true) cho số đẹp giả (87→7). Luôn chờ pending=0 rồi so cũ⟂mới trong CÙNG phiên Play. Đã thử heuristic "nhắm tâm region kế tiếp": tệ hơn trên núi (tâm region lồi lõm nằm sau vách).

Liên quan: [[project-walk-hitch-2026-10-17]] [[project-multifloor-pathfinding]] [[project-region-pathfinding]]
