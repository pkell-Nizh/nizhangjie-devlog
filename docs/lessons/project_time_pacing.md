---
name: project-time-pacing
description: "Nhịp thời gian chốt 2026-08-03 — ngày 18 phút thật, 30 TPS/32400 tick/ngày, lịch 4 tháng × 20 ngày; 3 số PHẢI đi đôi"
metadata: 
  node_type: memory
  type: project
  originSessionId: 26f62d4c-c99c-46da-9ff1-e286ff86df0b
  modified: 2026-08-03T10:20:25.409Z
---

**Chốt 2026-08-03** sau khi user báo *"game hơi vội, cả ngày chỉ đi làm rồi đi về là hết ngày, còn thời gian đâu mà sống"*. ĐÃ CODE, CHƯA biên dịch (Unity MCP bridge chết cả session).

## Bốn con số, và chúng KHÔNG độc lập

| Số | Ở đâu | Cũ → Mới | Điều khiển cái gì |
|---|---|---|---|
| `realSecondsPerDay` | `Data/time/time_config.json` (×2 bản) | 360 → **1080** | ngày dài bao nhiêu giây THẬT (cảm giác) |
| `TickRate.TicksPerSecond` | hằng số C# | 10 → **30** | độ mịn máy móc phản ứng theo tick (`OnFastTick`), KHÔNG phải độ mượt di chuyển |
| `TickRate.TicksPerHour` | hằng số C# | 150→450→**1350** | ⇒ `TicksPerDay` 3600→10800→**32400** (giữ đúng tỉ lệ TPS) |
| `GameDate` lịch | hằng số C# | 12×30=360 → **4×20=80** ngày/năm | tiến trình dài hạn |

**LUẬT PHẢI NHỚ: `TicksPerDay = realSecondsPerDay × TicksPerSecond`.** Thương của hai số = **số tick mỗi giây THẬT** = nhịp mô phỏng, phải giữ đúng bằng `TicksPerSecond`. Đổi độ dài ngày hoặc TPS mà quên đổi số kia thì sim âm thầm thô đi/mịn lên đúng tỉ lệ đó — **không lỗi, không log, chỉ là mọi hệ theo nhịp trượt**. Đã dính thật 2 lần trong cùng buổi: (1) kéo ngày 360s→1080s mà giữ 3600 tick ⇒ 3,33 tick/giây ⇒ `OnSlowTick` bỏ lỡ gặp gỡ NPC; (2) sau khi sửa (1) vẫn còn SÓT tài liệu cũ ở `PawnNeedsSystem`/`DayNightController`/`WildProductionSystem`/`M23TimeTest` ghi số tick tuyệt đối cũ — phải grep toàn bộ `3600`/`10800` mỗi lần đổi hằng số này. `GameClock.ApplyConfig` tự log warning nếu lệch runtime, nhưng KHÔNG bắt được comment/test lệch.

## Vì sao 18 phút / 80 ngày (xem chi tiết lý luận đầy đủ trong lịch sử chat, tóm tắt ở đây)

- **`SimTime.DeltaTime = Time.deltaTime × Speed`** — tốc độ đi bộ neo vào thời gian THẬT, KHÔNG neo vào độ dài ngày. Kéo dài ngày không làm NPC lê bước, chỉ làm đồng hồ thôi phóng vun vút.
- Chỉ số quyết định = **đi bộ tốn bao nhiêu GIỜ IN-GAME**: ngày 6 phút cũ ⇒ đi làm điển hình (100 ô) mất 4,2 giờ — cả ngày chỉ đi đường, đó là gốc "sống vội". Ngày 18 phút ⇒ 1,4 giờ.
- **18 chứ không 24**: trùng nhịp giờ RimWorld (45s vs 41,7s) — M25/M27/M16 đều tune theo nhịp đó.
- **Lịch 4×20 = 80 ngày/năm**: 360 ngày cũ × 18 phút = 108 giờ thật/năm, quá nặng cho tu tiên đo bằng năm. 80 ngày ⇒ 24 giờ thật/năm (8 giờ ở 3x). **4 tháng** (không phải 8/10) vì **1 tháng = 1 MÙA** (mirror RimWorld 4 quadrum) — mùa bám thẳng `GameDate.Month`, không cần hằng số riêng.

## Vì sao nâng TPS 10→30 (quyết định RIÊNG, sau khi đã chốt 18 phút/80 ngày)

User hỏi 3 kịch bản trước khi chốt — mỗi cái dạy một bài học khác, **chỉ 1/3 thật sự phụ thuộc TPS**:

1. **"NPC 10 ô/giây có clip xuyên tường không?"** → KHÔNG áp dụng. Mọi di chuyển (`WildPlayerToken`/`BeastBehaviour`/`NpcLifeOps.Step` qua `CityNpcSystem.Update`) chạy `Update()`+`SimTime.DeltaTime`, per-frame liên tục, tách hẳn khỏi tick. **Tick không di chuyển ai cả** — TPS cao/thấp không đổi độ mượt.
2. **"Ô lửa không kịp phát hiện pawn bước lên?"** → hazard-tile chưa tồn tại trong code (grep xác nhận 0 kết quả); nếu làm, nên phát hiện theo path-đã-xác-nhận (như cách chặn tường hiện tại) chứ không lấy mẫu theo tick — TPS không sửa tận gốc kiểu bug polling này (nâng TPS chỉ thu hẹp cửa sổ mù, không triệt tiêu).
3. **"Cửa tự động cần 30-60ms thay vì 100-200ms?"** → **ĐÂY LÀ CA THẬT DUY NHẤT.** Nếu máy móc hook `OnFastTick` (bắn mỗi tick), độ trễ = 1/`TicksPerSecond` giây thật — khác hẳn `OnSlowTick` (luôn cố định 1 Hz bất kể TPS, vì nó *là* `% TicksPerSecond`). TPS thật sự chi phối `OnFastTick`.

**Quyết định:** modder viết máy móc theo tick (thay vì tự viết continuous-check riêng như `WildSceneInteraction.Update()`) sẽ khổ ở 10 TPS (100-200ms/bước, dễ lỗi). 30 TPS hạ còn 30-60ms mà **chi phí engine ≈0** — `GameClock.StepTick()` thân hàm chỉ vài phép so `%`+tăng biến; 0 subscriber `OnFastTick`/`TickPhase` hiện tại ngoài `M23TimeTest`. 60 đắt gấp đôi 30 mà không thêm lợi ích nào cho modder — không đáng. Đây là node đầu tiên trong session cho phép ai đó viết cơ chế theo-tick mà không phải tự lo continuous-check.

## Hệ quả rơi ra (KHÔNG đổi dù TPS 10 hay 30, vì tỉ lệ `TicksPerHour/TicksPerSecond` giữ nguyên = 45)

- `slowTicksPerHour` (needs) 15 → **45** ⇒ Rest/Hunger/Mood tụt mượt gấp 3, nới rộng vùng chết mọi ngưỡng — chính loại bug đã làm đỏ M27.
- `TicksPerDay` vốn khai "BẤT BIẾN SAU KHI SHIP SAVE" — đổi được **chỉ vì game chưa ship**.

## Chỗ phải sửa theo (đã làm) + chỗ CHƯA

- ✅ `LocalMapTimeBar.Season` — mùa = tháng trực tiếp (bản cũ chia `(month-1)/3` cho lịch 12 tháng sẽ ra Xuân suốt tháng 1-3, không bao giờ có Thu/Đông với lịch 4 tháng).
- ✅ `M23TimeTest` mốc 45 ngày: tháng 3 ngày 6 (số cứng cố ý, làm canary cho lần đổi lịch sau).
- ✅ `M23TimeTest` dòng `ApplyConfig` — đổi từ hardcode `360f` sang **`TickRate.TicksPerDay / 10f`** (công thức, không phải số) để `TickLengthRealSeconds` luôn = 0.1s bất kể TPS/TicksPerDay sau này đổi thế nào — sửa 1 chỗ, miễn nhiễm cả 5 test khác (3/4/4b/5/6) đang advance bằng literal `0.1f`. Đây là lần thứ 2 dính đúng loại bug "số phải đi đôi" (`PawnNeedsSystem`/`DayNightController`/`WildProductionSystem` cũng bị sót từ lượt đổi TicksPerHour trước, tìm ra bằng grep `\b3600\b\|\b10800\b` toàn repo).
- ⚠ **CHƯA sửa, là số CÂN BẰNG chứ không phải hằng số lịch:** `SectRankPromotion._debounceCounter = 30` ngày và `WorldEventSystem.RollIntervalDays = 30` — vốn nghĩa "khoảng một tháng", giờ thành 1,5 tháng.

**Bài học quy trình:** mỗi lần đổi hằng số tick/lịch → BẮT BUỘC `grep -rn "\b<số cũ>\b" --include=*.cs .` toàn repo trước khi coi là xong, không chỉ sửa nơi vừa nghĩ tới. Comment mô tả con số tuyệt đối (không dùng `TickRate.X` symbolic) là nợ kỹ thuật tự tạo ra đúng lớp bug này.

Liên quan: [[project_city_npc_brain]] (hệ gặp gỡ lấy mẫu theo giây thật, không phụ thuộc TPS), [[feedback_data_folder_split]] (time_config.json có 2 bản).
