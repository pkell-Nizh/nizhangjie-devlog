---
name: project-cave-system
description: "Hệ hang nhiều loại 2026-10-21 (663d3fa4, nhánh worldgen-3d) — hang lớn, buồng cầu dẹp, khe rãnh, nước/dung nham ngầm theo ô có vách ngăn; tham số + bẫy"
metadata:
  node_type: memory
  type: project
  originSessionId: 62ee1e22-0d21-42dc-b346-287ab08d6fa3
  modified: 2026-10-20T17:29:27.691Z
---

User 2026-10-21: "hệ thống hang … hang hình cầu dẹp, khe rãnh dài, hệ thống lava, nước, các hang cực lớn rộng". Commit 663d3fa4.

- Code `World/Voxel/WorldGen3D.Caves.cs` (partial), `CarveChunk` thay bước (4) cũ của `WorldGen3D.Apply`. Mỗi nguồn hang ĐÁNH DẤU vào bộ đệm cột, rồi một lượt ghi không khí + chất lỏng ⇒ mọi nguồn cùng tuân luật vách ngăn.
- JSON `voxel/worldgen_3d.json` (2 bản): `caverns` (vùng hang lớn noise 2D + noise 3D), `chambers` (ô 48, 35%), `ravines` (ô 192, 30%, nửa mở lên mặt), `caveFluids` (ô 96 uốn: nước 25% mực 70–150, dung nham 20% mực 14–50, còn lại dung nham dưới 10).
- Bất biến chính: hai ô chất lỏng khác mực ⇒ VÁCH ĐÁ `barrier` khối mỗi bên tới mực cao hơn ⇒ không tường nước đứng, nước không chạm dung nham. Kiểm tra bằng CaveFluidAt trên cặp cột kề (luôn đúng nhờ ranh ô thẳng trục trong toạ độ đã uốn, độ uốn nhỏ).
- Khe mở lên mặt: ô gần nước/cây/công trình (SculptGuard bán kính 2) giữ mái `surfaceRoof` ⇒ thành cầu tự nhiên, không cây lơ lửng. Guard tính theo hình ⇒ ở ranh ô world có thể lệch một chút (chấp nhận, như điêu khắc).
- Test `Run Cave Tests` 17. Play Mode 25 ô: 0 tường chất lỏng, console sạch. Chi phí vùng hang lớn +18% (đo nóng; lần đầu JIT trông như ×2,5 — đừng tin số chạy lạnh).
- Chưa làm (chưa ai yêu cầu): trang trí hang (thạch nhũ, rêu), thác dung nham/nước, biome hang, miệng hang trên đất phẳng.

Liên quan [[project-worldgen-3d-redesign]], [[project_fluids]], [[project_sculpt_pillars_fix]].
