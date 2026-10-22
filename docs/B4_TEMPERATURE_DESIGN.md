# B4 — Nhiệt độ cơ thể + quần áo giữ nhiệt (2026-09-06)

> Trạng thái: **ĐÃ CODE + TEST 25/25 + VERIFY PLAY MODE**. Bản v1. Làm cho hệ Thời tiết "có sức nặng":
> nối Weather + needs + equipment.

## 1. Vì sao có

Hệ Thời tiết (WEATHER_SYSTEM) mới chỉ có tint + `moodPressurePerHour` nhẹ. Chưa có nhiệt độ, chưa có
lý do mặc áo ấm. B4 thêm: nhiệt độ môi trường (weather + ngày/đêm) → nhiệt độ **cảm nhận** (trừ cách
nhiệt trang bị) → ngoài dải dễ chịu thì đè Mood.

## 2. Kiến trúc — "1 con số áp lên toàn map" (mirror Weather/DayNight)

| Thành phần | File | Vai trò |
|---|---|---|
| `WeatherDefinition.tempOffsetC` | `Data/Definitions/WeatherDefinition.cs` | độ lệch nhiệt (°C) của loại thời tiết: quang 0, mưa −3, bão −7 |
| `EquipmentDefinition.insulationColdC` / `.insulationHeatC` | `Data/Definitions/EquipmentDefinition.cs` | cách nhiệt lạnh/nóng (°C), cộng dồn mọi món đang mặc. 0 = mặc định (vũ khí/trang sức/đồ cũ) |
| `TemperatureConfig` | `Core/Environment/TemperatureConfig.cs` | JSON `Data/environment/temperature_config.json`: baseTempC 20, dayPeak +6@15h, nightLow −8@5h, comfort 15..28°C, moodPerDegreePerHour 0.4, maxMoodPerHour 8, enableComfortPenalty |
| `TemperatureOps` | `Core/Environment/TemperatureOps.cs` | THUẦN: `DayNightSwingC(hour)` (cosine, nối vòng qua nửa đêm), `AmbientC(weatherOffset, hour)`, `FeltC(ambient, coldIns, heatIns, min, max)`, `MoodPressurePerHour(felt, ...)`, `Label` |
| `AmbientTemperature` | `Core/Environment/AmbientTemperature.cs` | STATIC "gương đọc" (như `WeatherController.GrowthMultiplierNow`): `CurrentC` = base + `WeatherController.Instance.CurrentWeather.tempOffsetC` + swing theo `GameClock` giờ. An toàn không có scene (thiếu clock → 12h). Config cache. |
| Hook | `World/Wild/Needs/PawnNeedsSystem.MoodPressureThisTick` | cộng thêm `TemperatureOps.MoodPressurePerHour(feltC, ...)` — mirror term thời tiết đã có. `SumApparelInsulation(pawn)` cộng `insulationColdC/HeatC` qua `EquipmentManager.GetAllEquipped(pawnId)` + `EquipmentDatabase.Resolve(item.OutputId)`. |

- **`FeltC`**: cách nhiệt LẠNH chỉ giúp khi ambient < comfortMin; NÓNG chỉ khi > comfortMax; và KHÔNG
  kéo vượt biên dải (áo ấm không làm ta thấy nóng khi ngoài trời −10°C, chỉ kéo lên tối đa tới
  comfortMin).
- **KHÔNG có MonoBehaviour mới** — B4 chỉ là data + ops + 1 hook vào vòng lặp Mood đã chạy sẵn mỗi
  slow-tick. Không đụng AI/pathfinding.
- **KHÔNG nối RoomDetector** ("mái nhà chắn rét") — BFS mắc, gọi mỗi pawn mỗi tick là phí (mirror lý do
  Weather không nối). v1.1.

## 3. Nội dung ship

- `Data/environment/temperature_config.json` (+ SA).
- `tempOffsetC` trên 3 weather: clear 0 / rain −3 / storm −7.
- `insulationColdC` trên 10 áo giáp (slot Armor): Thô Ma Y 2, Áo Vải Đay 2, Áo Da 3, Yêu Bì Giáp 6,
  Thiết Giáp 2, Ngọc Lân Giáp 3, Vân Ti Bào 2, Lân Vệ 3, Thiên Mạc 5, **Phượng Nhung Bào 9** (lông
  phượng = ấm nhất). Món chưa khai → 0 (không giữ nhiệt).

## 4. CỐ Ý CHƯA làm (v1)

- **Hạ thân nhiệt / say nắng thành BỆNH** (health condition có severity, ngất, chết) — v1 chỉ đè Mood.
  Cần thêm loại condition vào `HealthRegistry`.
- **Phòng giữ nhiệt / lò sưởi / máy lạnh** — RoomDetector đắt per-tick; v1 "1 số toàn map".
- **Mùa** (E1) — dao động nhiệt hiện chỉ theo GIỜ, chưa theo mùa/vĩ độ.
- **Weather NÓNG** (nắng gắt, `tempOffsetC` dương) — content sau, frame đã sẵn (`incident_cold_snap` của
  E-framework cũng có thể có bản "heat_wave").
- **Cách nhiệt theo slot đầy đủ** (mũ/ủng/găng) — v1 chỉ áo giáp có khai; field trên MỌI
  `EquipmentDefinition` nên thêm chỉ là sửa JSON.
- **Linh thú / pet** nhiệt độ — hook chỉ ở pawn `MoodPressureThisTick`; `StepCompanions` gọi
  `MoodPressureThisTick` chung nên PET CŨNG bị nhiệt độ (không có trang bị → luôn "trần"), chấp nhận
  được v1.

## 5. Test + verify

- `Run Temperature Tests` — **25/25** (`DayNightSwingC` tại đáy/đỉnh/giữa + liên tục qua nửa đêm;
  `AmbientC`; `FeltC` trong dải / lạnh có-cách-nhiệt / cách-nhiệt-thừa-cap-tại-min / cách-nhiệt-lạnh-vô-
  dụng-khi-nóng / cách-nhiệt-nóng; `MoodPressurePerHour` trong dải 0 / 5° dưới min = 2 / 10° trên max =
  4 / cực đoan clamp 8; `Label`; `TemperatureConfig.Load`+`Sanitize` swap dải đảo; `AmbientTemperature`
  không scene không throw + sane range).
- Hồi quy: Weather 30/30, M32 Mood 43/43, Needs (M25) 11/11, M34 EquipmentSlot 58/58.
- **VERIFY PLAY MODE 2026-09-06** (`WildLocal.unity`, pawn thật, qua `PawnNeedsSystem.MoodPressureThisTick`
  + `EquipmentManager` thật):
  - `AmbientTemperature.CurrentC`: điều kiện thường ~13°C; ép bão + 5h sáng → **đúng 5.0°C**
    (base 20 − low 8 − storm 7).
  - Pawn "Đệ Tử Cam" (no/nghỉ đầy, chỉ còn weather + nhiệt độ), 5°C:
    - không giáp → MoodPressure/giờ = **10.00** (weather storm 6 + nhiệt `(15−5)×0.4=4`).
    - mặc Phượng Nhung Bào (insul 9) → cảm nhận 14°C → **6.40** (nhiệt còn 0.4).
    - đổi Thô Ma Y (insul 2) → cảm nhận 7°C → **9.20** (nhiệt 3.2).
  - Áo ấm giảm khổ vì rét đo được, đi đúng qua hook thật. 0 error.
