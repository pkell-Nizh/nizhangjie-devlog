---
name: project_capture_tool
description: "Hệ CÔNG CỤ THU PHỤC (bóng/cầu) data-driven cho Pokémon pack + mọi mod — CaptureToolDefinition + fact-bag (loài/mục tiêu/MÔI TRƯỜNG, kể cả extraStats mod) + condition động; IP-safe (không móc data ball nào — Showdown items.ts CHỈ isPokeball:true, 0 công thức)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 1241bc70-cff5-4a7d-a328-4a86b7415309
  modified: 2026-09-09T12:12:22.260Z
---

**2026-09-09.** Nối tầng "loại bóng khác nhau" vào công thức bắt đã có (`CaptureOps.CatchChance` vốn sẵn param, chỉ đổi tên `ballMult`→`toolMult`). Generalize để mod có **quái riêng + thuộc tính riêng + bóng riêng** cũng chạy, 0 dòng engine.

## Đã đọc data ball — KHÔNG có gì để pull
`_PokemonReference/pokemon-showdown/data/items.ts`: mỗi ball chỉ `{name, spritenum, num, isPokeball:true}` — **0 công thức bắt** (Showdown không làm bắt thú hoang). Hệ số nhân (Great ×1.5 / Ultra ×2 / Net ×3.5 Water·Bug / Dusk ×3 đêm·hang / Quick ×5 lượt đầu / Timer ramp cap 4 / Nest (41−lvl)/10 / …) là **cơ chế** — reimplement từ Bulbapedia, không phải biểu đạt có bản quyền. Tên "Poké Ball"/sprite = TM → không ship.

## Kiến trúc — fact-bag phẳng + predicate trong def của tool
`Scripts/Core/Pawns/Capture/`:
- **`CaptureToolDefinition : Def`** (`Data/capture_tools/*.json` ship + `Data/Dev/capture_tools/*.json` demo — vắng trong public thì `LoadFromFolder` bỏ qua êm). `namespace:id` stamp ⇒ mod override/add. Field: `baseMultiplier`, `ruleCombine` (best|product|first), `guaranteedCatch` (master-ball phẳng), `oneUse`, `colorHex`/`visualPrefabAddress` (engine dựng khối đen; mod cấp art), `rules[]`. `ConfigErrors` bắt op lạ / rule không có allOf lẫn conditionId.
  - `CaptureToolRule` = `{ allOf: CaptureToolClause[], conditionId, param, multiplier, multiplierMax, guarantee }` — HOẶC `allOf` (AND clause dữ liệu) HOẶC `conditionId` (1 `ICaptureCondition`).
  - `CaptureToolClause` = `{ fact, op, value }`. op: eq·ne·lt·lte·gt·gte·present·absent. `present`/`absent` xét key tồn tại; op số so `_num`; eq/ne với key chuỗi so `_str`.
- **`CaptureFactBag`** — `Dictionary<string,float> _num` + `Dictionary<string,string> _str` + `HashSet<string> _flags`. `Set`/`SetString`/`Flag`/`Has`/`TryNum`/`TryStr` + `Eval(clause)`. Key có tiền tố quy ước `species.` / `target.` / `env.` nhưng engine chỉ tra Dictionary (KHÔNG reflection — hợp "string key thay enum" CLAUDE.md §7).
- **`CaptureContext` struct** — `{ species, hpFraction, level, statusId, turnIndex, biomeTag, timeOfDay, weather, alreadyCaughtSpecies }`.
- **`ICaptureFactProvider` + `CaptureFactRegistry`** (đăng ký TƯỜNG MINH, mirror `AbilityEffectRegistry`). `InstallCore()` ở GameRoot §6h-ter → đăng ký **`CoreCaptureFacts`**: đổ `species.capture_rate/danger_band/element`, cờ `species.apex`, `species.family:<x>`, `species.type:<t>`, **+ mọi kv `BeastSpeciesDefinition.extraStats` → `species.<key mod tự đặt>`** (đây là chỗ quái/thuộc tính MOD chạy không cần C#); `target.hp_fraction/level`, cờ `target.status[:id]`; `env.turn_index`, chuỗi `env.biome/time_of_day/weather`, cờ `env.already_caught_species`. Mod có hệ môi trường riêng (chu kỳ trăng…) đăng ký thêm provider.
- **`ICaptureCondition` + `CaptureConditionRegistry`** (đăng ký tường minh, `Get` strip `core:`/`capture_cond_`). `InstallCore()` → 3 lõi: `core:ramp_by_turn` (Timer: 1+step·(turn−1), param=step 0.3, cap `multiplierMax`|4), `core:ramp_by_low_hp` (1+k·(1−hpFrac)), `core:level_scaled` (Nest: (T−lvl)/10 clamp[1,cap], T=param|41). Trả HỆ SỐ (≤0 = không áp dụng).
- **`CaptureToolOps`** — lazy-load 2 folder, `Find` (id đầy đủ/trần), `IsOneUse`, `ResolveMultiplier(toolId, in ctx, out guaranteed)` (dựng bag → duyệt rules → combine best/product/first; `guaranteedCatch` hoặc rule `guarantee:true` → out true), `ValidateAllLoaded` (GameRoot), `ResetCacheForTests`. `BasicToolId = "capture_tool_basic"`.

## Nối vào game
`CorePawnInteractions` verb `ThrowCaptureObject`: `PickCaptureTool(actor)` = mục capture-tool đầu tiên trong `PawnBody.ItemBag` rồi `PlayerItemBag.Get()`; không có → `capture_tool_basic` (khối đen, `oneUse:false`, không trừ). `BuildCaptureContext` móc `timeOfDay` từ `DayNightController.Instance.HourOfDay` (6–17 day / 17–20 dusk / else night), `weather` từ `WeatherController.Instance.CurrentWeatherId` (strip `core:`/`weather_`), `level` từ `CreatureLevelRegistry`. `toolMult = CaptureToolOps.ResolveMultiplier(...)` → `CaptureOps.CatchChance(cfg, rate, hpFrac, 1f, toolMult)` + `guaranteed` bỏ qua xúc xắc. Trừ 1 tool nếu `IsOneUse`. **v1 chưa làm**: `biomeTag` (cần Cell), `alreadyCaughtSpecies`, `turnIndex` (=1 ngoài trận theo lượt), UI chọn tool, art khối đen thật (mới `colorHex` trong def).

## Data
Ship: `Data/capture_tools/capture_tool_basic.json` (×2 folder, ×1, `oneUse:false`, rules rỗng). Demo `Data/Dev/capture_tools/` (8, KHÔNG vào public build): sturdy(×1.5) / aqua(type water·bug ×3.5) / dusk(night|cave ×3) / swift(turn≤1 ×5) / timer(ramp_by_turn) / nest(level_scaled) / **deepsea (VÍ DỤ MOD: `species.aquatic`+`species.abyssal` từ extraStats, `ruleCombine:product` → 2×2.5)** / master(guaranteedCatch).

## Test — `Run Capture Tool Tests` 32/32
Khắc hệ loài, môi trường (giờ/biome), lượt đầu, condition ramp, level-scale clamp, master guaranteed, **mod species extraStats + product combine**, tool lạ→×1, `toolMult` nâng `CatchChance`, fact-bag present/absent/số, ConfigErrors sạch. Hồi quy: Creature Team 42/42 (gồm CaptureOps), M64 TurnBased 242/242, Pokemon Arena 20/20.

Liên quan: [[project_m64_combat_mode_framework]] (P4c nhánh Pokémon), [[project_modding_strategy]], [[feedback_data_folder_split]].
