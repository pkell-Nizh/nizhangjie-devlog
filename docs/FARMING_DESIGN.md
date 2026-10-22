# Thiết kế TRỒNG TRỌT tổng quát (2026-10-23) — ĐÃ CHỐT (§11), chưa code

> Mục tiêu: một đường ống duy nhất cho MỌI thứ "mọc" trên khối — cây trồng, mầm cây, bụi quả, dược thảo, nấm,
> rong, mía/tre, bí/dưa, linh thảo của pack xianxia — để thêm loài mới = thả một file JSON, thêm luật mới =
> đăng ký một handler. Không fork theo từng loài (luật A6).

---

## 0. Hiện trạng (đã đọc code, đừng làm lại)

| Thứ đã có | Ở đâu | Còn thiếu |
|---|---|---|
| Cây trồng là **khối có nấc** (`khối@n`), tự vào save | `BlockDefinition.stages` | — |
| Nhịp ngẫu nhiên kiểu MC randomTick, handler theo chuỗi | `World/Voxel/BlockTicks` (`grow`, `sapling`) | `grow` chỉ tung xác suất theo `growDays`, **không xét ánh sáng/nước/đất/nhiệt** |
| Mất chỗ đỡ ⇒ bật ra vật phẩm | `BlockTicks.OnBlockReplaced` + `needsSupport` | — |
| Chỉ mọc trên khối nào | `plantOn` (danh sách **id khối**) | id cứng ⇒ thêm loại đất mới phải sửa mọi cây |
| Hái khi chín ⇒ về nấc `harvestToStage` | `BlockPlacementOps.Break` | một món + một khoảng số; không có hạt giống trả lại, không theo công cụ |
| Đặt hạt = đặt khối | tool_action `core:place_block` | — |
| Vùng Trồng + pawn tự gieo | `JobGiver_GrowPlant`, `WildZoneOps` | gieo trên cỏ/đất trần; **hái vẫn phải Designate tay**; cây chọn qua `MapFeatureDefinition crop_*` (**dữ liệu trùng** với `block_crop_*`, di sản 2D) |
| Ánh sáng khối 0..15 | `VoxelWorld.LightAt`, `Chunks.Light` | chưa ai đọc cho cây |
| Nhiệt độ môi trường | `Core/Environment/AmbientTemperature` | toàn cục, chưa theo vị trí/biome |
| Chất lỏng (nước nguồn/chảy) | `FluidFlow`, `IBlockTickWorld.FluidAt` | — |

Kết luận: **khung đã đúng hướng** (khối + nấc + randomTick + handler registry). Việc là thêm *điều kiện*,
*kết cục*, *đất*, *thu hoạch*, rồi nối AI — không dựng hệ mới.

---

## 1. Mô hình: "cây" = khối có `plant`

Gom mọi field cây đang nằm rải trên `BlockDefinition` vào **một object con `plant`** (`PlantSpec`). Khối không
có `plant` ⇒ không phải cây. Field cũ (`growDays`, `plantOn`, `harvest*`, `growsIntoTree`) vẫn đọc được: loader
tự dựng `PlantSpec` từ chúng nếu JSON không khai `plant` ⇒ 86 file khối + mod cũ không phải sửa.

```jsonc
// block_crop_wheat.json (minh hoạ)
{
  "id": "block_crop_wheat", "itemId": "wheat_seeds", "solid": false, "stages": 8, "tick": "plant",
  "plant": {
    "growDays": 0.4,                       // trung bình ngày/nấc khi mọi điều kiện = 1
    "soil": ["farmland"],                  // TAG khối đỡ, không phải id
    "factors": [                           // điều kiện mọc — xem §2
      { "id": "core:light",  "min": 9 },
      { "id": "core:soil" },               // độ màu mỡ + độ ẩm của khối đỡ
      { "id": "core:temperature", "min": 5, "max": 32 },
      { "id": "core:crowding" }            // trồng xen hàng nhanh hơn ruộng kín một loại (MC)
    ],
    "onMature": { "id": "core:stay" },     // kết cục — xem §3
    "harvest": {                           // thu hoạch — xem §4
      "fromStage": 7, "toStage": -1,       // -1 = cây một vụ, hái xong mất khối
      "drops": [
        { "item": "wheat", "min": 1, "max": 1 },
        { "item": "wheat_seeds", "min": 0, "max": 3 }
      ],
      "unripeDrops": [ { "item": "wheat_seeds", "min": 1, "max": 1 } ]
    }
  }
}
```

Handler tick mới **`plant`** thay cho `grow`/`sapling` (hai cái cũ giữ làm bí danh trỏ về `plant`).

---

## 2. Điều kiện mọc = danh sách FACTOR (cắm thêm được)

Mỗi lần nhịp ngẫu nhiên trúng cây:

```
chance = ChancePer(growDays) × Π factor_i.Multiplier(ctx, params_i)
```

- Factor trả **0** = chặn hẳn (thiếu sáng, chết rét). Trả **>1** = mọc nhanh hơn (đất tốt).
- `IGrowthFactor` + `GrowthFactorRegistry` — **mirror `BlockTicks.Register`**: đăng ký tường minh, không
  reflection, id có namespace. Factor lạ trong JSON ⇒ `ConfigProblems` báo + coi như 1 (không khoá cây vì 1 chữ
  gõ sai — cùng lý lẽ `EquipmentSlotGates`).
- Factor là **hàm thuần** đọc `BlockTickContext` (thêm `LightAt`, `FluidAt`, khối đỡ) ⇒ test không cần Play Mode.

Bộ factor lõi (đủ cho MC + RimWorld):

| id | Đọc gì | Ghi chú |
|---|---|---|
| `core:light` | `max(sáng trời theo giờ, sáng khối)` ở ô cây | `min`/`max`; nấm khai `max: 12` (thích tối) |
| `core:sky` | ô có thấy trời không | cây cần mưa/nắng thật; trong nhà kính đèn thì bỏ factor này |
| `core:soil` | `fertility` + nấc ẩm của khối đỡ (§5) | luật MC: đất ướt ×~2 so đất khô |
| `core:temperature` | `AmbientTemperature` (sau này theo biome/độ cao) | ngoài dải ⇒ 0; ngoài xa ⇒ cây chết (`onFail`, §3) |
| `core:water_adjacent` | có nước trong N ô ngang | mía, lúa nước |
| `core:submerged` | ô cây đang ngập chất lỏng | rong, sen — khối khai `holdsFluid` |
| `core:crowding` | 8 ô quanh cùng loại | bản MC: hàng xen kẽ nhanh hơn |
| `core:biome` | biome tại khối | `allow` / `bonus` theo id biome |

**Điểm cắm cho pack**: xianxia đăng ký `xianxia:lingqi` (đọc linh khí ô world) — linh thảo chỉ mọc ở vùng linh
khí dày. Gỡ pack ⇒ factor biến mất ⇒ linh thảo mất điều kiện đó (factor lạ = 1), game không sập. Đúng phép thử
A5. Mùa (sau này, E1) cũng chỉ là một factor `core:season`, không đụng đường ống.

---

## 3. Kết cục khi lên nấc cuối = `onMature` (cắm thêm được)

Cùng kiểu registry (`IPlantOutcome`). Bộ lõi:

| id | Hành vi | Loài |
|---|---|---|
| `core:stay` | đứng ở nấc chín chờ hái | lúa, rau, bụi quả |
| `core:tree` | dựng cây (`TreeGrowth.TryGrow`, có sẵn) | mầm cây |
| `core:grow_up` | đặt thêm một khối cùng loại phía trên, trần `maxHeight` | mía, tre, xương rồng |
| `core:fruit_beside` | đặt khối quả (`fruitBlock`) vào ô ngang trống có đất hợp | bí, dưa (thân còn lại, quả hái riêng) |
| `core:spread` | lan sang ô kề hợp điều kiện | cỏ, nấm, dây leo |

Thêm `onFail` tuỳ chọn (factor trả 0 liên tục / nhiệt ngoài xa): `core:wither` ⇒ đổi sang khối cây héo
(`witheredBlock`) hoặc bật ra hạt. Mặc định **không chết** (MC) — RimWorld-style thì pack tự khai.

---

## 4. Thu hoạch thống nhất

- `harvest.drops` / `unripeDrops` = bảng rơi nhiều món (thay `harvestItemId + min/max` đơn lẻ). Đọc bằng một
  hàm `PlantOps.RollDrops(spec, stage, tool)` — mọi bên (người chơi, pawn, máy móc sau này) đi qua đó.
- `toStage`: ≥ 0 = cây lâu năm về nấc đó (bụi quả); `-1` = một vụ, mất khối, đất còn lại (lúa).
- **Công cụ** theo tag (`EquipmentDefinition.power`, hệ có sẵn): `sickle` ⇒ cộng sản lượng (`toolBonus`) và/hoặc
  gặt 3×3 (`areaWithTag`). Không có liềm vẫn gặt được bằng tay.
- Đập cây chưa chín ⇒ `unripeDrops` (thường trả lại 1 hạt) — sửa luôn việc hiện giờ đập rau non mất hạt hay
  không là do `itemId` quyết định một cách tình cờ.
- Giữ nguyên hành vi hiện có: chín thì "đập = hái" ở `BlockPlacementOps.Break`.

---

## 5. Đất = khối có `soil` (tag + màu mỡ + độ ẩm)

`BlockDefinition.soil` (object con, tuỳ chọn):

```jsonc
"soil": { "tags": ["soil", "tillable"], "fertility": 1.0, "tillsTo": "block_farmland" }   // block_dirt / block_grass
"soil": { "tags": ["soil", "farmland"], "fertility": 1.0, "moistureStages": true,          // block_farmland
          "hydrateRange": 4, "dryTo": "block_dirt", "trampleTo": "block_dirt" }
```

- **Cây khai `soil: [tag]`** thay cho `plantOn: [id]` ⇒ mod thêm "đất linh" chỉ cần khai tag `farmland` +
  `fertility: 2` là MỌI cây trồng tự mọc trên đó, không sửa file cây nào. `plantOn` cũ vẫn đọc (khớp id).
- **Đất ruộng** `block_farmland`: 2 nấc khô/ướt (hoặc 0..7 như MC). Tick `soil`: có nước trong `hydrateRange` ô
  (`FluidAt`) hoặc **đang mưa và thấy trời** ⇒ ướt; khô lâu mà không có cây ⇒ về `dryTo`. Nhảy/rơi lên ⇒
  `trampleTo` (tuỳ chọn, tắt được trong config).
- **Cuốc** = tool_action mới `core:till`: chuột phải bằng món có tag `hoe` lên khối có tag `tillable` và ô trên
  trống ⇒ đổi sang `tillsTo`. Data: `wep_wood_hoe`/`wep_stone_hoe`… + công thức.
- **Phân bón** = tool_action `core:fertilize` (bột xương / phân / tro): ép N lần tick `plant` **bỏ qua xác suất**
  nhưng **vẫn tôn trọng factor = 0** (bón phân trong hang tối không làm lúa mọc). Món khai `fertilizeTicks`.

---

## 6. AI: Vùng Trồng làm trọn vòng

`JobGiver_GrowPlant` mở rộng thành **"làm ruộng"**, vẫn Tier 3, capability `core:plants`, theo thứ tự ưu tiên
trong một vùng: **hái chín → gieo ô trống có đất ruộng → cuốc ô chưa cuốc**. (Hái trước để ruộng không đứng
chín mãi; cuốc cuối vì cần công cụ.) Mỗi bước là một `Toils.WorkUntil`, gọi đúng hàm thuần mà người chơi gọi
(`PlantOps.Till/Plant/Harvest`) — một luật cho cả hai, như `HarvestOps`.

- Vùng chọn **hạt giống** (vật phẩm) thay vì `MapFeatureDefinition crop_*`: danh sách = mọi khối có `plant` và
  `itemId` là vật phẩm. **Gỡ 2 file `feat_crop_*`** (dữ liệu trùng, di sản 2D — hướng đã chốt: gỡ dần
  phụ thuộc world 2D).
- Hạt lấy từ túi pawn → kho gần nhất (như `JobGiver_Haul`). Không có hạt ⇒ bỏ qua bước gieo, không kẹt.
- Sản phẩm vào túi pawn, đầy thì rơi đất (luật `BlockPlacementOps.Give` có sẵn); khuân về kho là việc của hệ
  khuân, không viết riêng.

---

## 7. Nguồn hạt giống (để vòng chơi khép kín)

- Đập cỏ cao / cỏ dại (`block_tall_grass`, khối mới) ⇒ `dropChance` ra hạt (MC).
- Cây dại sinh sẵn theo biome (cà rốt/khoai dại) ⇒ đã có cơ chế rải cây cỏ khi gen.
- Thu hoạch trả hạt (`drops` §4). Mua ở cửa hàng (M65) — chỉ là data.

---

## 8. Vùng ngoài tầm (ĐÃ CHỐT: A, trần 10 ngày — xem §11)

Nhịp ngẫu nhiên chỉ chạy ở chunk đang nạp (giống MC). Game có pawn làm ruộng ở xa thì ruộng **đứng yên** khi
người chơi đi xa. Hai lựa chọn:

- **A (đề xuất) — bù lúc nạp lại:** chunk nhớ ngày cuối được nạp; nạp lại thì mỗi khối cây được bù số lần tick
  theo phân phối nhị thức với số ngày đã trôi (trần ví dụ 10 ngày), dùng điều kiện tại thời điểm nạp. Rẻ (chỉ
  quét khối có `plant` trong chunk vừa nạp), ruộng xa vẫn chín khi quay về.
- **B — như MC:** đứng yên. Đơn giản nhất, nhưng mâu thuẫn với "pawn sống thật" đã chốt ([[feedback_all_creatures_full_physics]]).

---

## 9. Lưu / hiệu năng

- Mọi trạng thái (nấc cây, độ ẩm đất) nằm trong id khối ⇒ **save miễn phí**, không IO mới.
- Factor đọc ánh sáng/chất lỏng theo ô: rẻ, nhưng tick `soil` quét `hydrateRange`² ô — **chỉ quét khi đất
  khô** (đã ướt thì kiểm thưa), giống MC. Đo lại `Ticks.Random` sau khi làm (ngân sách 1,5 ms/khung có sẵn).
- Mục 8A: chỉ chạy lúc nạp chunk, rải qua `BudgetedQueue` nếu đo thấy giật.

---

## 10. Thứ tự làm (mỗi bước chơi được)

> **Bước 1 XONG (2026-10-23).** Khác bản thiết kế ở 2 chỗ:
> - Tag đất nằm ở `BlockDefinition.tags` có sẵn (không thêm `soil.tags`). `soil` chỉ còn giữ `fertility`; độ ẩm thêm ở bước 2.
> - `core:light` không theo giờ (như MC: `max(trời, khối)`). Đêm không làm cây ngừng; chỉ chỗ tối thật mới ngừng.
>
> **Bước 2 XONG (2026-10-23).** Đất ruộng 2 nấc (khô/ướt). Cây đọc độ ẩm lúc tính factor, nên mưa có tác dụng ngay.
> Hoãn lại: giẫm hỏng ruộng, texture theo nấc. Pawn cuốc chưa cần cầm cuốc (để bước 5).
>
> **Bước 3 XONG (2026-10-23).** Đã có bảng rơi, cây một vụ, liềm (cộng sản lượng + gặt 3×3), lúa mì / cà rốt / khoai tây, bánh mì, khoai nướng.
> - Thêm luật ăn ⟂ gieo: món ăn được mà gieo được chỉ gieo khi ngắm đúng chỗ trồng.
> - Chưa có nguồn hạt ngoài thế giới; bước 4 làm cỏ cao.
>
> **Bước 4 XONG (2026-10-23).** Đã có `grow_up` (mía), `fruit_beside` (bí), `spread` (nấm), factor `water_adjacent`, cỏ cao rơi hạt và sinh lúc gen thế giới.
> - Khối mới có cờ `replaceable`: cuốc hoặc gieo thì dọn đi.
> - Sửa kèm: cây chỉ bật ra khi luật đỡ thật sự hỏng (mía chồng mía không đổ khi khối dưới đổi nấc).
>
> **Bước 5 XONG (2026-10-23).** Pawn làm ruộng trọn vòng (gặt → gieo → lấy hạt ở kho), vùng chọn hạt giống, đã gỡ `feat_crop_*`.
> - Khác bản thiết kế: cuốc nằm trong việc gieo, không phải một việc riêng; pawn chưa cần cầm cuốc.
> - Ruộng ngoại ô ghi thẳng id khối cây; ô Farmland có mặt là `block_farmland`.
>
> **Bước 6 XONG (2026-10-23).** Bù tăng trưởng khi ô world nạp lại (trần 10 ngày, sổ `SectorSeenStore` vào save). Phân bón: Phân Ủ, Bột Xương.
> - Sửa kèm: một lần gọi nhịp có thể lên nhiều nấc. Trước đó xác suất bị chặn ở 1 nên cây nhanh chậm hơn thiết kế.
> - Bù chỉ xét khối trong sổ sửa (cây người trồng / đã hái). Điều kiện mọc xét lúc nạp, không mô phỏng mưa lúc vắng.

1. `PlantSpec` + factor registry + handler `plant` + 4 factor `light/soil/temperature/crowding`; loader dựng
   spec từ field cũ. Test thuần.
2. Đất: `soil` trên khối, `block_farmland`, tick ẩm, `core:till` + cuốc gỗ/đá + công thức. Cây trồng đổi sang
   `soil: ["farmland"]`.
3. Thu hoạch: `drops`/`unripeDrops`/`toStage: -1`, liềm. Nội dung: lúa mì, cà rốt, khoai tây (thêm vào rau +
   linh điền thảo có sẵn); bánh mì ở bếp.
4. Kết cục `grow_up` + `fruit_beside` + `spread`; mía, bí, nấm, cỏ cao ra hạt.
5. AI làm ruộng trọn vòng; gỡ `feat_crop_*`.
6. Bù tăng trưởng vùng ngoài tầm (nếu chọn 8A). Phân bón.
7. (Pack xianxia) `xianxia:lingqi` + đất linh + linh thảo.

Ngoài phạm vi (đợt riêng): chăn nuôi/giao phối, mùa vụ (E1), chất lượng nông sản theo tay nghề.

---

## 11. Quyết định của user (chốt 2026-10-23)

1. **Vùng ngoài tầm: chọn 8A.** Khi nạp lại thì bù phần tăng trưởng đã lỡ, trần **10 ngày**
   (`plant_config.json → catchUpMaxDays`, sửa được không cần code).
2. **Cuốc hay không là do từng loài tự khai, không có luật chung.**
   - Khối cây khai `plant.soil` theo tag: `["farmland"]` thì phải cuốc trước, `["soil"]` thì gieo thẳng lên cỏ/đất.
   - Hạt giống / mầm non (vật phẩm) trỏ tới khối cây nó đặt ra. Đặt hạt xuống thì kiểm luật đất của khối cây đó.
     Vì vậy UI và AI biết trước một hạt có cần đất ruộng không, khỏi phải thử đặt.
   - Ví dụ: lúa mì, cà rốt cần cuốc; bụi quả, dược thảo, mầm cây thì không.
3. **Điều kiện xấu: lõi chỉ dừng lớn** (factor = 0 thì đứng yên, không chết).
   - `onFail` / `core:wither` vẫn nằm trong registry, nhưng không loài lõi nào khai.
   - Pack nào muốn héo chết thì tự khai.
4. **Mưa tưới ruộng.** Tick `soil`: đang mưa và ô đất thấy trời thì đất ướt, y như có nước trong `hydrateRange`.
   - Loại thời tiết khai cờ `wetsSoil` (mưa, bão bật; trời quang tắt). Mod thêm "mưa phùn" chỉ cần khai cờ này.
