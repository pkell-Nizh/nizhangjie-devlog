---
name: project-biome-redesign
description: "Thiết kế lại biome cho thế giới 3D (2026-10-20) — biome tính tại khối từ trường địa hình, cỡ MC; thú/sự kiện đọc theo chunk; xianxia ra mod pack; chấp nhận vỡ save"
metadata:
  node_type: memory
  type: project
  originSessionId: 62ee1e22-0d21-42dc-b346-287ab08d6fa3
  modified: 2026-10-19T22:05:08.273Z
---

User 2026-10-20: biome cũ là nhãn dán trên ô world 2D (240 khối, quyết định TRƯỚC địa hình 3D) ⇒ lỗi thời, cần thiết kế lại. Nhánh `worldgen-3d`.

**Đã chốt:**
- Core = engine tổng quát; bộ biome mặc định là địa lý trái đất thật. Xianxia là mod, không thuộc core (mình từng quên — xem [[feedback-core-sim-before-xianxia]]).
- Biome = hàm tại từng khối từ cùng trường đã sinh địa hình: nhiệt, ẩm, lục địa, xói mòn, đỉnh–thung, tính đất, độ cao thật (nhiệt giảm theo độ cao ⇒ tuyết theo núi thật).
- Cỡ mảng biome kiểu Minecraft (vài trăm khối).
- Thú/sự kiện đọc biome THEO CHUNK nơi chúng sinh/xảy ra (user: "người chơi ở đâu cũng đâu quan trọng"), không theo chỗ người chơi, không theo nhãn ô world.
- Nhãn ô world 2D = biome chiếm nhiều nhất trong ô (suy ra từ hàm, không ngược lại).
- Chấp nhận vỡ save.
- Tiên Cảnh / Ma Vực / `lingQiBase` / SpecialZone theo linh mạch ⇒ chuyển sang mod pack xianxia ngay đợt này, qua cơ chế "vùng phủ" tổng quát bằng JSON của core.

**Thứ tự (mỗi bước 1 commit):** 1 hàm biome + lớp tool bản đồ · 2 nhãn world từ hàm mới · 3 luật mặt đất + tuyết theo độ cao · 4 hình địa hình từng biome (mục D cũ) · 5 cơ chế vùng phủ + dời xianxia · 6 biome hang · 7 nội dung biome mới.

Liên quan [[project-worldgen-3d-redesign]].

**Tiến độ:**
- ✅ Bước 1 (2026-10-20): `World/Terrain/BiomeField` + `BiomeDefinition.climate` (`BiomeClimate`/`ClimateRange`) + `Data/terrain/biome_field.json` + 25 biome core có climate (16 file mới) + lớp "Biome mới" trong tool bản đồ + test Biome Field 17. Chưa nối game. ⚠ `Fill` ~93 ms/ô world (C# thường) — cần Burst hoặc lấy mẫu thưa hơn trước bước 2. ⚠ Test tạo biome giả "t_*" ⇒ intern vào BiomeRegistry trong phiên Editor (vô hại).
- ✅ Đợt A1 (2026-10-20): khí hậu NOISE THUẦN — `BiomeField` không nhận `WorldMap` nữa; user chốt lạnh/nóng rải khắp nơi kiểu MC, KHÔNG dải vĩ độ. Fbm → hàm phân phối chuẩn (sd 0,2005) ⇒ nhiệt/ẩm phân bố đều, khoảng JSON ≈ tỉ lệ diện tích. Config: temperatureBias/humidityBias thay worldTemperatureMin/Max. Test 19. Còn lại của A: hệ hỏi biome theo chunk (thú/sự kiện/dân số/HUD) + tăng tốc Fill (Burst).
- (Cũ, hết hiệu lực sau A1) Bẫy: nhiệt world = vĩ độ thuần (tương quan −0,98 với y), kẹt 0 cả dải cực ⇒ `worldTemperatureMin` 0,06 để dải đó không toàn đồng tuyết.

**Hướng dài hạn (user 2026-10-20: "sau này phải gỡ 2d ra cho nhẹ, bị phụ thuộc nhiều không tốt"):** mục tiêu là BỎ phụ thuộc vào bản đồ world 2D 512² (WorldMap/WorldGenerator) ở runtime. Mọi thứ mới phải là HÀM THEO TOẠ ĐỘ + seed (kiểu Minecraft), tính lười theo chunk/vùng. KHÔNG dán nhãn lên ô world nữa, KHÔNG thêm hệ mới đọc `Cell`. Bước 2 cũ ("nhãn world từ hàm mới") bị BỎ; thay bằng: BiomeField tự sinh khí hậu bằng noise (không đọc Cell), các hệ hỏi biome theo chunk.
