# Làm mod cho Nizhangjie

Mod chia làm hai tầng, hứa hẹn khác nhau:

| Tầng | Cách làm | Ổn định? |
|---|---|---|
| **Dữ liệu** | Thả file JSON vào `Mods/<mod>/Data/` | **Có.** Tên field và tên thư mục là hợp đồng công khai. |
| **Móc C#** | Viết code đăng ký vào các sổ móc của lõi | **Không hứa.** API có thể đổi giữa các bản. |

Phần lớn mod chỉ cần tầng dữ liệu. Không có Lua hay script nhúng.

---

## 1. Cấu trúc thư mục

```
StreamingAssets/Mods/
  my_mod/
    mod_info.json
    Data/
      beasts/        ← cùng tên thư mục với StreamingAssets/Data/
      traits/
      rebirth_rules/
      ...
```

- Loader chỉ quét **thư mục con có `Data/`**. File để thẳng ở gốc `Mods/` (như `_EXAMPLE_mod_info.json` và file này) không bao giờ được nạp.
- Tên thư mục con trong `Data/` phải **trùng đúng** tên thư mục ở `StreamingAssets/Data/` (vd `beasts`, `traits`, `wild_buildings`). Tên sai thì file bị bỏ qua, không báo lỗi.

## 2. `mod_info.json`

```json
{
  "id": "my_mod",
  "displayName": "Mod của tôi",
  "version": "1.0.0",
  "author": "tên bạn",
  "description": "Mô tả ngắn.",
  "enabled": true,
  "dependencies": ["core"],
  "loadAfter": [],
  "loadBefore": []
}
```

| Field | Ý nghĩa |
|---|---|
| `id` | Namespace của mod. Tự chuyển về chữ thường, chỉ giữ `[a-z0-9_]`. **Nên trùng tên thư mục** (lệch thì có cảnh báo, `id` thắng). `core` là id dành riêng. |
| `enabled` | `false` = cất mod đi mà không xoá. Mod phụ thuộc vào nó cũng tự tắt theo. |
| `dependencies` | Bắt buộc. Thiếu mod nào trong đây thì mod này bị loại (không làm hỏng mod khác). `core` luôn có sẵn. |
| `loadAfter` / `loadBefore` | Mềm: chỉ ảnh hưởng thứ tự nếu mod kia có mặt. |

**Thứ tự nạp:** lõi trước, rồi các mod theo thứ tự phụ thuộc (hoà thì theo abc). Mod dính vòng phụ thuộc bị loại cả nhóm, có log lỗi rõ.

## 3. Id và tiền tố (kiểu Minecraft)

Trong JSON bạn chỉ viết **id cục bộ**, không kèm tiền tố:

```json
{ "id": "fire_fox", "displayName": "Hồ Lửa", ... }
```

Loader tự gắn namespace. Với phần lớn kiểu dữ liệu mới (kế thừa `Def`), khoá thật trong game là **`my_mod:fire_fox`**.

**Tham chiếu sang def khác:**
- Có dấu `:` → dùng nguyên văn: `"core:beast_wolf"`, `"pokemon:move_tackle"`.
- Không có `:` → tìm trong **chính mod của bạn** trước, rồi `core:`.

⇒ Tham chiếu tới nội dung của mod khác thì **luôn ghi đủ tiền tố**, và khai mod đó trong `dependencies`.

### Thêm mới hay ghi đè?

Tuỳ kiểu dữ liệu:

- **Kiểu `Def` (có tiền tố)** — `beasts`, `races`, `rebirth_rules`, `capabilities`, `moves`, `abilities`, `types`, `natures`, `injuries`, `status_effects`, `thoughts`… Cùng id cục bộ với lõi **không đè** lõi: `my_mod:beast_wolf` và `core:beast_wolf` là hai con khác nhau. Đây là **thêm mới**.
- **Kiểu cũ (id trần)** — một số thư mục chưa chuyển sang `Def` (vd `traits`, `materials`, `equipment`, `wild_buildings`…). Ở đây cùng id là **ghi đè**: mod nạp sau thắng, log có dòng `Override ID '...'`.

Không chắc kiểu nào? Nạp game, xem log: có `Override ID` là đè, không có là thêm.

## 4. Kiểm tra lỗi lúc khởi động

Sau khi nạp, lõi chạy bước kiểm tra toàn bộ def:
- Tham chiếu trỏ tới def không tồn tại → lỗi.
- Giá trị ngoài miền (vd `hpFraction` ngoài (0,1], `packMin > packMax`) → lỗi.

Mọi lỗi được gom thành **một báo cáo** trong Console, ghi rõ def nào, field nào. File JSON hỏng cú pháp chỉ làm mất file đó (`[JsonDataLoader] Lỗi parse ...`), không làm sập mod.

## 5. Sổ điều kiện chung

Nhiều hệ nhận danh sách `conditions` (tiến hoá, trait ẩn, hồi sinh, công cụ thu phục…). Cùng một cú pháp ở mọi nơi:

```json
"conditions": [
  { "id": "min_level", "value": "16" },
  { "id": "time_of_day", "value": "night" },
  { "id": "fact", "param": "count.kills", "op": "gte", "value": "100" }
]
```

Tất cả phải đạt (AND). Chưa đạt thì game hiện lý do cho người chơi.

| id | Dùng | Ghi chú |
|---|---|---|
| `min_level` / `min_bond` | `value` = số | bond 0..100 |
| `min_age` / `max_age` | `value` = năm | |
| `time_of_day` | `day` \| `dusk` \| `night` | |
| `weather` | `clear` \| `rain` \| `storm` … | |
| `biome` | id vùng | |
| `race` | id chủng tộc | có/không tiền tố đều được |
| `gender` | `0` nam · `1` nữ · `2` không giới tính | |
| `knows_skill` | id chiêu | |
| `has_trait` | id trait | |
| `has_item` | id vật phẩm | |
| `fact` | `param` = khoá, `op` = `gte` `gt` `lte` `lt` `eq` `ne` `present` `absent`, `value` | so **bất kỳ** khoá nào trong túi dữ kiện |
| `flag` | `param` = tên cờ | đúng khi cờ đã bật |

**Khoá chuẩn trong túi dữ kiện:** `level`, `bond`, `gender`, `age`, `race`, `life_stage`, `species`, `env.time_of_day`, `env.weather`, `env.biome`, `skill.<id>`, `trait.<id>`, `count.<tên>` (bộ đếm theo pawn, vd `count.kills`).

`fact` + `flag` đủ cho phần lớn mod chỉ viết JSON. Cần luật mới hẳn thì xem §7.

## 6. Các hệ thường mod

### Loài sinh vật — `beasts/`
Xem mẫu: `Mods/pokemon/Data/beasts/species_bulbasaur.json`. Trỏ tới chỉ số chiến đấu qua `combatUnitId` (thư mục `combat_units/`).

### Trait ẩn — `traits/`
```json
{
  "id": "trait_sat_tinh",
  "displayName": "Sát Tinh",
  "isHidden": true,
  "unlockConditions": [ { "id": "fact", "param": "count.kills", "op": "gte", "value": "100" } ],
  "modifiers": []
}
```
Pawn tự mở trait khi đạt điều kiện; trước đó UI không hiện.

### Chủng tộc + giai đoạn đời — `races/`
Khai `lifeStages` (mốc tuổi → id giai đoạn) và `warnYearsBeforeEnd`. Mỗi ngày in-game pawn già thêm (80 ngày = 1 năm), qua mốc thì đổi giai đoạn, hết tuổi thọ thì chết già. Mẫu: `StreamingAssets/Data/races/`.

### Hồi sinh — `rebirth_rules/`
**Lõi không ship luật nào — mặc định chết là chết thật.** Mod bật hồi sinh bằng cách khai luật:
```json
{
  "id": "phoenix_rebirth",
  "kinds": ["pawn", "companion"],
  "races": ["phoenix"],
  "conditions": [ { "id": "min_level", "value": "20" } ],
  "cooldownDays": 3,
  "hpFraction": 0.5,
  "priority": 0
}
```
- `kinds`: `pawn` \| `companion` (quái hoang không hồi sinh). Rỗng = cả hai.
- Điều kiện được xét trên dữ kiện **lúc chết**.
- Nhiều luật khớp → `priority` cao thắng.
- Hàng chờ hồi sinh được lưu vào save.

Mẫu thật: `Mods/tensura/Data/rebirth_rules/named_soul_rebirth.json`.

## 7. Móc C# (không hứa ổn định)

Mod có code chạy qua một lớp "content pack" gọi `Install` một lần lúc khởi động, và **tự kiểm tra mod có đang bật không**:

```csharp
if (!Nizhangjie.Modding.ModLoader.IsLoaded("my_mod")) return;
```

Nhờ vậy tắt mod = sửa `enabled` trong `mod_info.json`, không phải sửa code. Mẫu đầy đủ: `Scripts/Pokemon/PokemonContentPack.cs`.

| Sổ | Để làm gì |
|---|---|
| `ConditionRegistry.Register(new FnCondition("my_mod:x", (bag, c) => (ok, lýDo)))` | Thêm loại điều kiện mới. **Giữ tiền tố mod** trong id. |
| `CreatureHooks.Register(ICreatureLifecycleHook)` | Nghe mốc vòng đời sinh vật: sinh ra, có chủ, lên cấp, tiến hoá, thắng trận, bị gỡ. Kế thừa lớp no-op rồi ghi đè đúng mốc cần. |
| `CreatureHooks.Register(ICreatureFactProvider)` | Đổ dữ kiện riêng của mod vào túi (vd cờ `my_mod:blessed`) để JSON `flag`/`fact` dùng được. |
| `CreatureHooks.Register(ICreatureAppearance)` | Vẻ ngoài đặc biệt (màu đè, nhãn sau tên). |
| `PackSaveRegistry.Register(PackSaveParticipant<T>)` | Lưu dữ liệu riêng của mod. Khoá **có tiền tố** (`"my_mod:state"`). Mod bị tắt thì mục của nó vẫn được giữ trong save, bật lại không mất. |
| `GlobalEventBus.Subscribe<CreatureDiedEvent>` | Nghe mọi cái chết (pawn, quái, linh thú). Chỉ báo tin, không huỷ được. Muốn "đỡ chết" pawn thì nghe `DiscipleDiedEvent` và `Cancel()`. |

## 8. Quy ước

- Id: chữ thường, `snake_case`, tiếng Anh không dấu. `displayName`/`description` viết tiếng gì cũng được.
- File JSON lưu **UTF-8 không BOM**.
- Mỗi def một file, tên file = id.
- Đừng đặt tên, hình ảnh hay roster lấy từ thương hiệu có bản quyền vào mod định phát hành.
