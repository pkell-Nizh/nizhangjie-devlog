# Group E — Frame "Người kể chuyện cấp thế giới" / World Incident (2026-09-06)

> Trạng thái: **ĐÃ CODE + TEST 32/32 + VERIFY PLAY MODE**. Bản v1. Là FRAME data-driven cho Group E
> ("Thế giới sống động & sự kiện") — E4 (đoàn buôn), E5 (đột kích), E7 (sự kiện thời tiết) là
> **content** cắm vào frame này, mỗi cái 1 file JSON + 1 `IIncidentHandler`.

## 1. Vì sao có — và khác Event Pool thế nào

Đã có **Event Pool** (`EventPoolDefinition` + `WildEventTriggerSystem`): roll MÙ theo bảng biome/danger
**PER-Ô-BẢN-ĐỒ** (encounter table kiểu Pokémon). Cái còn thiếu cho "thế giới sống động" là sự kiện
cấp **COLONY** quyết định theo **ĐIỀU KIỆN** (bao lâu kể từ lần trước loại này, ngày thứ mấy) rồi nhắm
một **nhịp trung bình** — đúng mô hình "Storyteller / IncidentDef" của RimWorld.

| | Event Pool (có sẵn) | World Incident (E-framework, mới) |
|---|---|---|
| Phạm vi | 1 ô bản đồ | Cả colony / thế giới |
| Chọn kiểu gì | Roll mù, bảng weighted theo biome | Lọc theo điều kiện (cooldown/ngày) rồi bốc weighted |
| Nhịp | Mỗi lần vào ô mới + mỗi N ngày | Nhắm 1 nhịp trung bình (dao động [0.5×, 1.5×]) |
| Cooldown | Không | Riêng từng incident (`minDaysBetween`) |
| Xác định | `DetHash` (replay được) | `System.Random` + lưu state (mirror Weather) |

**HAI HỆ SONG SONG, không thay nhau** — "gặp gì trên đường" ⟂ "chuyện lớn của cả colony".

## 2. Kiến trúc

| Thành phần | File | Vai trò |
|---|---|---|
| `IncidentDefinition` | `Data/Definitions/IncidentDefinition.cs` | JSON `Data/incidents/*.json`: `category`, `weight`, `minDaysBetween` (cooldown riêng), `minGameDay`, `handlerId`, `stringParam`+`floatParam` (tham số đơn giản, KHÔNG dict — JsonUtility). |
| `IncidentOps` | `World/Wild/Incidents/IncidentOps.cs` | THUẦN: `IsEligible` / `Eligible` (lọc + sort id) / `PickWeighted(roll01)` / `NextInterval(mean, roll01)`. |
| `IIncidentHandler` + `IncidentHandlerRegistry` | `World/Wild/Incidents/IIncidentHandler.cs` | Mirror `WildEventHandlerRegistry` / `ShopBehaviorRegistry`: đăng ký tường minh theo `handlerId`, KHÔNG reflection; handlerId lạ → incident bị loại khỏi vòng bốc (`IsEligible` false), không throw; exception trong handler bị nuốt + log. |
| `CoreIncidentHandlers` | `World/Wild/Incidents/CoreIncidentHandlers.cs` | 3 handler LÕI (mỗi cái 1 ca thật, luật D6). |
| `WorldIncidentDirector` | `World/Wild/Incidents/WorldIncidentDirector.cs` | Plain C#, ServiceLocator (`Resolve()`/`ReplaceAll()`, mirror `ResearchRegistry`/`QuestRegistry`). State: `lastFiredDay` per-id + `lastAnyDay` + `nextTargetDay`. `Tick(gameDay, rng, meanDaysBetween, bootstrap)`. `ForceRun` (debug/quest). |
| `WorldIncidentSystem` | `World/Wild/Incidents/WorldIncidentSystem.cs` | VỎ MonoBehaviour, `GameClock.OnDayTick`, `LocalMapBootstrap.EnsureMapSystems` tự gắn. `_meanDaysBetweenIncidents=2.5`, `_enabled`. Director + state TOÀN CỤC (ServiceLocator + IncidentIO), chỉ VỎ này per-map. |
| `IncidentIO` + `SaveData.incidents` | `SaveSystem/IncidentIO.cs` | Capture/Restore, wired `SaveManager` 3 chỗ (mirror `ResearchIO`). Save cũ → director tự lên lịch lại. |
| Đăng ký | `Core/Bootstrap/GameRoot.cs` §6d-octies | `ServiceLocator.Register(new WorldIncidentDirector())`. |

## 3. Vòng đời 1 nhịp `Tick`

1. `nextTargetDay < 0` → lên lịch `gameDay + NextInterval(mean, rand)`, return.
2. `gameDay < nextTargetDay` → chờ, return.
3. Tới hạn: `Eligible(defs, gameDay, lastFiredDay, handlerExists)` → `PickWeighted(rand)` → ghi
   `lastFiredDay[id]` + `lastAnyDay` → lên lịch lần kế → `IncidentHandlerRegistry.Run(ctx)`.
4. Không có incident nào đủ điều kiện → `nextTargetDay = gameDay + 1` (thử lại mai, không spin).

## 4. Nội dung ship (`Data/incidents/`, 4 file, 2 bản) — 3 handler LÕI

| id | category | weight | cooldown | minDay | handler | hiệu ứng |
|---|---|---|---|---|---|---|
| `incident_quiet_night` (Đêm Yên Tĩnh) | misc | 5 | 1 | 0 | `core:flavor` | chỉ log |
| `incident_wandering_stranger` (Người Lạ Đi Ngang) | visitor | 3 | 3 | 0 | `core:flavor` | chỉ log (đoàn buôn THẬT = E4 sau) |
| `incident_cold_snap` (Rét Đậm) | weather | 2 | 6 | 2 | `core:force_weather` | ép `WeatherController` → `core:weather_storm` 8 giờ |
| `incident_resource_windfall` (Trúng Mối) | boon | 2 | 5 | 1 | `core:resource_cache` | +8±4 đơn vị 1 material trung tính vào `PlayerItemBag` chung |

`core:force_weather` **tái dùng hệ Thời tiết** (WEATHER_SYSTEM_DESIGN). `core:resource_cache` mirror
`World.Events.CoreWildEventHandlers` ResourceFind. `core:flavor` = "nothing happens" weighted cao để
incident không dồn dập.

## 5. CỐ Ý CHƯA làm (v1)

- **Spawn thật** (đoàn buôn E4, party đột kích E5) — cần spawn NPC pawn/party trên map, engine chưa có
  API tổng quát; `core:flavor` giữ chỗ, content-pack thật tự thêm handler (luật A5).
- **Nhịp riêng theo category** (RimWorld: threat budget ⟂ boon budget) — v1 chỉ 1 nhịp chung.
- **Điều kiện phong phú hơn** (dân số, độ giàu, mùa, thời tiết hiện tại) — v1 chỉ cooldown + minGameDay.
  Thêm field vào `IncidentDefinition` + check trong `IncidentOps.IsEligible` là mở rộng thẳng.
- **Hệ thư báo / notification UI (E6)** — v1 chỉ `Debug.Log`. Mọi handler log 1 dòng, gắn UI sau chỉ
  cần đọc log/event.
- **Storyteller "tính cách"** (RimWorld Cassandra/Phoebe/Randy) — chọn `meanDaysBetween` + weight
  scaling theo profile. v1 hằng số.
- **City/Sect** — vỏ tick per-map Wild (`LocalMapData` là khái niệm Wild).

## 6. Test + verify

- `Run Incident Tests` — **32/32** (`IncidentOps`: eligible/cooldown/minGameDay/handler-missing/
  weight-0 filter/sort-by-id/PickWeighted cumulative/NextInterval biên; `IncidentHandlerRegistry`: 3
  handler, unknown no-throw, `core:resource_cache` cấp 8-14 đơn vị thật, `core:force_weather` no-throw
  khi thiếu Instance; `WorldIncidentDirector` end-to-end 4 incident thật: nhịp đầu chỉ lên lịch, fire
  tại/sau target, ghi 1 lần, reschedule, `ForceRun` ghi cooldown, `ReplaceAll` restore + cooldown vẫn
  chặn sau restore).
- Hồi quy: Research (D5) 37/37, M53 Memory Save 34/34, M61 BuildPersist 44/44, WildEvent+Storyteller
  16/16.
- **VERIFY PLAY MODE 2026-09-06** (`WildLocal.unity`, qua GameRoot + EnsureMapSystems thật):
  - `WorldIncidentDirector` trong ServiceLocator, `WorldIncidentSystem` attached, `IncidentHandlerRegistry.Count==3`.
  - Tua `GameClock` ~32 ngày qua `OnDayTick` thật → incident fire đều nhịp ~2.5 ngày: ngày 3 Người Lạ
    → ngày 6 Đêm Yên Tĩnh → ngày 9 Người Lạ (cooldown 3 ngày tôn trọng) → ngày 12 Trúng Mối `+14
    oak_log vào túi` → ... Weighted đúng (Đêm Yên Tĩnh weight 5 xuất hiện nhiều nhất).
  - `ForceRun("incident_cold_snap")` → `core:force_weather` handler đổi `WeatherController` rain→storm
    THẬT, cooldown ghi @ ngày 32.
  - Save round-trip qua `IncidentIO.Capture` → `JsonUtility` → `ReplaceAll` → 4 loại + lastAnyDay +
    nextTargetDay khôi phục đúng. 0 error.
