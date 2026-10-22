---
name: project-sculpt-pillars-fix
description: "Cột cỏ/đá vô tri trên thế giới 3D (2026-10-20) do điêu khắc 3D KHOÉT quanh mặt đất gốc — sửa bằng giới hạn theo đất lân cận; cách đo trong Play Mode"
metadata:
  type: project
---

User 2026-10-20 báo "nhiều cột đá khá vô tri". Nguyên nhân đo được: điêu khắc 3D (`sculpt*`, commit 56a86e65) khoét 5–7 khối quanh một cột ⇒ mặt đất gốc đứng lại thành cột phủ cỏ (34/37), mọc thêm chỉ 3/37. Hàm độ cao không có lỗi.

Sửa (nhánh worldgen-3d): `VoxelWorld.SculptLimits` — chỉ nặn trong [đất thấp nhất, đất cao nhất] bán kính `sculptSupportRadius`=2 ± `sculptSlack`=1. Kết quả 0 cột nhô.

**Bẫy khi đo:** quét "khối đặc cao nhất" phải LOẠI khối cây (log/leaf/wood) — lần đầu mình tưởng 525 cột/ô do mọc thêm, thật ra là cây. Đo trong Play Mode qua `VoxelWorld._sectors` → `Chunks` (VoxelChunkSet) `Get`/`Base.TerrainLayers`. Mô phỏng ngoài game bằng `SculptNoise/SculptSolid` KHÔNG bắt được (phải xem khối thật).

**Why:** biên độ điêu khắc ~14×1,6 khối ≫ độ dốc mặt đất ⇒ noise áp đảo, đất phẳng bị đục thành đảo/cột.
**How to apply:** chỉnh điêu khắc thì giữ luật giới hạn theo lân cận; hố sâu còn lại là miệng hang (`worldgen_3d.json caves.entrance*`), hệ riêng. Liên quan [[project-worldgen-3d-redesign]].
