# Hệ Thời Tiết — thiết kế (2026-09-05)

> Trạng thái: **ĐÃ CODE + TEST XANH (30/30) + VERIFY PLAY MODE**. Bản v1.

## 1. Vì sao có

Trước đợt này, "Ngày/Đêm" (M24) là hạ tầng môi trường DUY NHẤT — chỉ có tint sáng-tối theo giờ,
KHÔNG có mưa/tuyết/nhiệt độ nào ảnh hưởng tinh thần pawn hay tốc độ tài nguyên. Đây là mảng
life-sim tiêu chuẩn (RimWorld-style) còn thiếu hẳn.

User chốt 2 yêu cầu:
1. **Data-driven** — mỗi loại thời tiết là data, không hardcode.
2. **Framework chỉnh sửa được** — modder thêm loại thời tiết mới không phải sửa code.

## 2. Mối lo đã giải: "đổi thời tiết có kích hoạt AI/pathfinding toàn bộ pawn cùng lúc không?"

**KHÔNG.** Thiết kế cố ý KHÔNG có hành vi "pawn tự chạy đi trú mưa". Nếu có, tất cả pawn cùng đọc
thấy trời mưa ở cùng 1 tick → cùng request pathfinding trong cùng 1 frame → giật khựng đồng loạt
(dù ngắn). Thay vào đó, 2 hiệu ứng gameplay đều là **SỐ THUẦN mà hệ khác TỰ ĐỌC khi tới lượt tick
của chúng**:

- **Mood** — `PawnNeedsSystem.MoodPressureThisTick` cộng thêm `WeatherController.MoodPressurePerHourNow`,
  TRONG vòng lặp per-pawn ĐÃ CHẠY SẴN mỗi slow-tick (M32). Không thêm lời gọi mới nào.
- **Tốc độ bù tài nguyên** — `ResourcePoolSystem.HandleDayTick` nhân `chance` với
  `WeatherController.GrowthMultiplierNow`, **1 lần/ngày** (không per-pawn, không per-tick).

Nếu sau này muốn thêm "pawn tìm chỗ trú", cách đúng là **rải ngẫu nhiên thời điểm mỗi pawn kiểm
tra** (offset theo hash `pawnId`, hoặc chỉ check khi pawn đang Idle/Tier thấp) — KHÔNG phải 1 sự kiện
toàn cục đánh thức tất cả cùng lúc. Để dành khi có nhu cầu thật (luật D6).

## 3. Vì sao KHÔNG dựng registry + interface kiểu `IShopBehavior`

2 hiệu ứng (Mood pressure, growth multiplier) đều là **biến thiên ĐỊNH LƯỢNG** (mạnh/yếu bao nhiêu),
không phải **hành vi ĐỊNH TÍNH** (làm việc gì khác nhau). Field thường trên `WeatherDefinition` giải
quyết gọn — mirror cách `ShopDefinition.buyMarkup`/`sellMargin` là field, còn `IShopBehavior` chỉ
dành cho hành vi thật ("khách vãng lai tự ghé mua"). Dựng registry+interface cho đúng 2 con số là
lặp lại bài học "IShopCurrencyAccount cho 1 currency" (luật D6).

**"Framework" nằm ở chỗ MỖI LOẠI THỜI TIẾT là 1 file JSON rời** (`Data/weather_types/*.json`, nạp
qua `DataRegistry<WeatherDefinition>.LoadFromFolder` — cộng dồn, mirror `ShopDefinition`/
`MaterialDefinition`). Modder thêm "sương mù"/"bão cát"/"nắng gắt" = thả 1 file, KHÔNG sửa code hay
file khác. Đây là điểm moddable **rộng hơn** `DayNightConfig` (bảng keyframe trong 1 file, không tách
được từng mốc).

## 4. Kiến trúc (mirror `DayNightController` từng nét)

| Thành phần | File | Vai trò |
|---|---|---|
| `WeatherDefinition` | `Data/Definitions/WeatherDefinition.cs` | 1 loại thời tiết: id, displayName, weight, min/maxDurationHours, tintColorHex, tintMaxAlpha, moodPressurePerHour, growthMultiplier |
| `WeatherRuntimeConfig` | `Core/Weather/WeatherRuntimeConfig.cs` | cấu hình TOÀN CỤC: enableWeather, activeScenes, smoothSeconds, maxOverlayAlpha. File `Data/weather/weather_runtime_config.json` |
| `WeatherOps` | `Core/Weather/WeatherOps.cs` | THUẦN, test không cần Play Mode: `PickWeighted(list, roll01)` + `RollDurationHours(def, roll01)` |
| `WeatherController` | `Core/Weather/WeatherController.cs` | MonoBehaviour, `GameRoot.Install()` gắn (mục 7-bis), tick theo `GameClock.OnHourTick`, overlay tint riêng |

- **Overlay RIÊNG** — `~WeatherOverlay`, sortingOrder **32001** (ngay trên `~DayNightOverlay` 32000).
  2 lớp tint chồng nhau: đêm mưa = tối (day/night) + xám xanh (weather). Cơ chế `EnsureOverlay`/
  `FitOverlayToCamera`/`ApplyVisual` là **copy line-for-line** của `DayNightController` (sprite trắng
  1×1 runtime, Unlit shader, bám camera, làm mượt bằng `UnityEngine.Time.deltaTime` — TRÌNH DIỄN nên
  đo giờ thực, luật M23.1).
- **Bốc thời tiết KHÔNG cần determinism** (mirror `ResourcePoolSystem`/`ForestLifecycleSystem` — sim
  SAU worldgen, không phải generator): dùng `UnityEngine.Random.value`. `WeatherOps` nhận `roll01` đã
  rút sẵn nên test được không cần Unity Random.
- Sort danh sách def theo id trước khi dựng dải trọng số (bẫy A4 — `DataRegistry.GetAll()` là
  Dictionary). Không bắt buộc cho determinism, chỉ cho ổn định/test được.

## 5. Nội dung ship kèm (`Data/weather_types/`)

| id | displayName | weight | duration (h) | tint | moodP/h | growthX |
|---|---|---|---|---|---|---|
| `core:weather_clear` | Trời Quang | 6 | 6–16 | không | 0 | 1.0 |
| `core:weather_rain` | Mưa | 3 | 3–9 | #5C6B82 α0.28 | 2 | 1.5 |
| `core:weather_storm` | Bão | 1 | 2–5 | #3A4254 α0.45 | 6 | 1.15 |

`activeScenes` mặc định chỉ `["WildLocal"]` — 2 hiệu ứng (ResourcePoolSystem, PawnNeedsSystem) hiện
chỉ có ở Wild. Rỗng = áp mọi scene (fail-open, mirror DayNightConfig).

## 6. CỐ Ý CHƯA làm (v1)

- **Không persist vào save.** Load lại → bốc thời tiết mới. Chấp nhận được: khác sử ký/ký ức (mất =
  tự mâu thuẫn), mất thời tiết chỉ là gián đoạn nhẹ về hình ảnh (đang bão → quang sau load). Mirror
  DayNight (cũng không persist, suy từ tick). `WeatherController.RestoreState(id, endsAtTick)` đã có
  sẵn làm seam cho `WeatherIO` sau này nếu cần (luật B5 — khai vòng đời dữ liệu tường minh).
- **Không có "pawn tìm chỗ trú".** Xem §2.
- **Không nối RoomDetector** ("mái nhà chắn mưa" → mood pressure chỉ áp pawn ngoài trời). Lý do:
  `RoomDetector.FindEnclosedRoom` là BFS tới 400 ô — gọi mỗi pawn mỗi slow-tick là phí rõ ràng.
  Mirror lý lẽ DayNight ("phòng kín cũng tối như ngoài trời"). Có sẵn `RoomDetector` nếu đợt sau
  cần.
- **Không có HUD hiển thị loại thời tiết** — chỉ overlay tint + log console (mirror M24 ship lần đầu).
- **Không nối City/Sect** — 2 hệ tiêu thụ hiệu ứng chưa có ở đó.
- **Không có work-speed multiplier ngoài trời** — Wild JobDriver chưa có hook "tốc độ làm việc"
  chung; thêm sẽ phải sửa nhiều JobGiver (ngoài scope v1).

## 7. Test + verify

- `Run Weather Tests` — **30/30** (accessor an toàn khi null Instance, config load/AppliesToScene,
  data 3 loại core nạp từ JSON thật, `PickWeighted` dải cộng dồn + weight 0 + all-zero + list rỗng/
  null, `RollDurationHours` biên + min>max swap).
- Hồi quy: M32 Mood 43/43, Needs (M25) 11/11, ResourcePool 14/14, M37 Decoupling 42/42
  (`Arch_CoreNeverDependsOnSim` xanh — `Core/Weather/` sạch Sim), DayNight 21/21.
- **VERIFY PLAY MODE (2026-09-05, `WildLocal.unity`)** qua đúng `GameRoot` thật:
  - Boot log liệt kê "Weather"; `[Weather] Bật ở scene "WildLocal"`; bốc `Trời Quang` ~7.5h, JSON
    đọc từ `Assets/Data/weather_types` (không fallback).
  - Ép `RestoreState("core:weather_rain")` → `MoodPressurePerHourNow=2`, `GrowthMultiplierNow=1.5`.
  - **Tích hợp Mood chính xác:** `PawnNeedsSystem.MoodPressureThisTick` cho pawn thật (no đủ/nghỉ
    đủ/không đau) — withRain − withClear = **0.04444** = đúng `2 / slowTicksPerHour`.
  - `~WeatherOverlay` dựng đúng: sortingOrder **32001** > DayNight 32000; sau khi ép Update() nhiều
    nhịp với rain → `enabled=True`, màu lerp về #5C6B82, alpha lerp về 0.28, scale khớp viewport;
    ép về clear → alpha mờ dần về 0.
  - 0 error/warning suốt phiên.
