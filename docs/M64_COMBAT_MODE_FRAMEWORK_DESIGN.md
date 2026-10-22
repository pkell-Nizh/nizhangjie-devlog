# M64 — COMBAT MODE FRAMEWORK (khung nhiều lối đánh)

> **Trạng thái:** Phase 0-4 ĐỀU CÓ KẾT QUẢ verify Play Mode 2026-09-02 → 2026-09-03 (xem §10 + §10.2/§10.3).
> Còn nợ: skill trong turn-based, arena nhiều-đấu-sĩ, scene arena riêng, đổi party. Bug `DamageCalculator`
> ×0.8 (có sẵn từ trước) đã spawn task riêng.
> **Người viết:** phiên làm việc 2026-09-02 → 2026-09-03.
> **Đọc kèm:** `CODE_SYSTEM_RULE.md` (A1 JSON-first, A4 Deterministic, A5 Lõi⟂Content, A6 Cấm fork),
> `M22_WILD_COMBAT_DESIGN.md`, `M_COMBAT_DESIGN.md`, `M_DATA_REGISTRY_DESIGN.md`.

---

## 1. Vấn đề

Yêu cầu user (nguyên văn, 2026-09-02):

> "mình muốn gameplay kiểu combat là phần tách hoàn toàn so với dữ liệu"
> "đợt này mình muốn tạo thêm hệ thống turnbase (pokemon style), hoặc là rất nhiều các thể loại kiểu gameplay khác"
> "bỏ [M22 realtime] qua một bên, tạo một tab option chọn loại combat lúc tạo file game lần đầu (hiện ta chưa làm giờ). mình muốn làm một framework để họ có thể chơi theo bất kỳ kiểu gì player thích"

Diễn giải: **không phải làm "một hệ combat thứ hai"**, mà là làm **khung** để:

1. Luật giao tranh (realtime-with-pause / turn-based Pokémon / … về sau) là **module cắm rời**, chọn được.
2. Lựa chọn đó là **quyết định lúc tạo ván chơi**, lưu vào file save, cố định cho ván đó.
3. Cùng **một bộ dữ liệu** (pawn, chỉ số, skill, ailment, vật phẩm) nuôi được **mọi** module.

Đây đúng tinh thần luật A5: *dự án là ENGINE, một lối đánh cụ thể chỉ là nội dung*. "Turn-based kiểu
Pokémon" là **cơ chế** (được phép nằm trong core-pack); "Cửu Dương Thần Công gây 200% sát thương hỏa" là
**nội dung** (JSON / xianxia-pack).

---

## 2. "Tách khỏi dữ liệu" nghĩa là gì ở đây

Hai trục riêng, làm cả hai nhưng trục A là phần khó:

| Trục | Nội dung | Phép thử sống-hay-chết |
|---|---|---|
| **A. Kernel combat không phụ thuộc NỘI DUNG** | Vòng giải quyết đòn không nhắc tên một skill / ailment / damage-type cụ thể. Tất cả nằm trong JSON. | Modder thêm 1 skill lạ → **không phải mở Visual Studio, không phải sửa kernel một dòng**. |
| **B. Sim không phụ thuộc COMBAT** | Bỏ nguyên module combat → colony-sim / hội thoại / kinh tế vẫn chạy. Combat là content-pack tuỳ chọn. | Xoá `Assets/Scripts/Combat/` + bỏ `CombatContentPack.Install()` → game vẫn đói/mệt/làm việc/chết. |

Trục B phần lớn ĐÃ đạt: `Combat/` đã là assembly riêng, `Core/` không phụ thuộc nó, `CombatRuntime` là
plain C#. M64 giữ nguyên trục B, chỉ siết thêm trục A + thêm lớp chọn mode.

---

## 3. Đã có sẵn gì

Soát `Assets/Scripts/Combat/` — bộ khung đã tồn tại từ M22. **Cột "M64 dùng thế nào" phân biệt rõ: cái
nào là hạ tầng/Model CHUNG (giữ), cái nào là nội thất RTwP (chỉ `RtwpCombatDriver` xài — driver khác KHÔNG
bắt buộc mượn, xem §4.4).**

| Thành phần | Vai trò | M64 dùng thế nào |
|---|---|---|
| `CombatRuntime` (plain C#) | Orchestrator 1 trận: encounter + RTwP + move + loot. | **Refactor**: thôi hardwire RTwP, uỷ thác phần "xếp lịch" cho `ICombatDriver`. Hạ tầng chung. |
| `CombatEncounter` (MonoBehaviour) | Vòng đời + điều kiện thắng (wipe team A/B). | **Giữ nguyên** — dùng chung mọi mode. Điều kiện thắng nâng thành `IVictoryCondition` nếu cần. |
| `CombatRTwPController` + `ActionGaugeSystem` + `CombatTickClock` | Mô hình thời gian RTwP (gauge, pause, tick). | **Bọc lại** thành `RtwpCombatDriver` — không sửa logic. |
| `CombatBehaviorSystem` | Bộ não AI quái, tick-paced. | **Nội thất RTwP.** Turn-based viết bộ chọn nước đi theo lượt MỚI cho khít — không bẻ cong cái này. |
| `SkillCastController` | Cast time đếm bằng tick. | **Nội thất RTwP.** Turn-based resolve trong lượt, không cần. |
| `GridMovementSystem` / `CombatGrid` / `GridFootprint` | Di chuyển + ô chiếm trên lưới. | **Nội thất RTwP.** Turn-based hàng-trước/sau không có lưới, không kéo vào (xem §6). |
| `ICombatSession` (interface) | Bề mặt UI bind vào — **đã thiết kế scene-agnostic** (harness / wildmap / dungeon). | **Mở rộng, không thay** — thêm cờ `UsesGrid`, `SupportsPartySwitch`. |
| `CombatPlayerController` | Input người chơi (chọn unit / target / skill). | RTwP driver sở hữu. Turn-based có controller input riêng. |
| `DeterministicRng` (seeded) | RNG tất định. | Mọi driver nhận từ `CombatRuntime` — luật A4. |
| `DataRegistry<T>` + JSON loader | `CombatUnitDefinition`, `SkillDefinition`, `CombatScenarioDefinition`. | Thêm `CombatModeDefinition`. |

**Nợ kỹ thuật phát hiện khi soát (KHÔNG thuộc phạm vi M64, ghi lại để xử riêng):**
`Combat/Ailments/AilmentDefinition.cs` là **ScriptableObject** `[CreateAssetMenu]` — **vi phạm luật A1**.
Song song còn `Combat/Elements/ElementalAilmentController` và `Combat/StatusEffects/`. Hệ ailment đang có
≥2 nhánh. M64 sẽ *chạm* vào ailment (cadence tick theo mode, §6) nên cần biết, nhưng dọn nhánh ailment là
việc riêng.

---

## 4. Kiến trúc đề xuất — 3 tầng

```
┌─ COMBAT MODEL ────────────────────────────────────────────────────────────┐
│  Chia sẻ mọi mode · ăn data · KHÔNG có khái niệm thời gian / lượt          │
│                                                                          │
│  Combatant (bọc PawnBody + snapshot chỉ số)                               │
│  Ability   (resolve từ SkillDefinition)                                   │
│  EffectPipeline → registry EffectWorker "namespace:id"                     │
│       core:damage · core:apply_ailment · core:knockback · core:heal ...    │
│       xianxia:qi_drain · xianxia:shield_break ...                          │
│  AilmentRuntime · Team · IVictoryCondition                                │
│  (tái dùng: CombatEncounter, DeterministicRng, CombatUnit, DamageCalculator)│
└──────────────────────────────────────────────────────────────────────────┘
        ▲                     ▲                          ▲
   ICombatDriver         ICombatDriver              ICombatDriver
   RtwpCombatDriver      TurnBasedCombatDriver      (sau: TacticalGrid,
   = M22 hôm nay,        vòng rời · initiative       AutoBattler, CardBattle…)
   bọc sau seam          theo Agility · 1 action/    → chỉ cần class mới +
                         combatant/vòng · slot       Register vào factory
                         phi-toạ-độ · đổi party ·
                         PP (actionCost)
        ▲
   Encounter (cầu nối THẾ GIỚI ↔ driver)
   SeamlessEncounter (Wild — không đổi scene, M22)
   ArenaEncounter    (snapshot người tham gia → chạy driver → ghi kết quả:
                      HP qua M30 · chết · loot · XP · ailment còn hiệu lực)
```

### 4.1 `ICombatDriver`

```csharp
public interface ICombatDriver
{
    void Attach(CombatEncounter encounter, DeterministicRng rng, CombatModeDefinition mode);
    void Begin(Team playerTeam);

    bool UsesGrid            { get; }   // UI/camera/scenario tự thích nghi
    bool SupportsPartySwitch { get; }   // turn-based bật; RTwP tắt

    // Người chơi VÀ AI đều đổ nước đi qua đây — 1 cửa
    void RequestAbility(string casterId, AbilityRequest request);

    void Tick(float deltaTime);         // RTwP dùng; turn-based có thể tự chạy coroutine, để trống

    ICombatSession Session { get; }     // UI bind — đã scene-agnostic
    event Action<Team> Ended;
}
```

### 4.2 `CombatDriverRegistry` (mirror list-composition của `BuildingFunctionRegistry`)

```csharp
CombatDriverRegistry.Register("core:rtwp",      () => new RtwpCombatDriver());
CombatDriverRegistry.Register("core:turnbased", () => new TurnBasedCombatDriver());
// xianxia-pack / mod gọi Register trong Install() của nó — KHÔNG sửa core
```

Luật A6: **mọi** mode đi qua `CombatRuntime` + `ICombatDriver`. Cấm dựng `CombatRuntime` thứ hai.
Luật A2: registry lấy qua `ServiceLocator`, không `Instance`.

### 4.3 `CombatModeDefinition` (JSON — luật A1)

`Assets/Data/combat_modes/*.json` + `StreamingAssets/Data/combat_modes/*.json` (nhớ 2 thư mục KHÔNG tự
đồng bộ — `feedback_data_folder_split`).

```jsonc
// core_rtwp.json
{
  "id": "core:rtwp",
  "displayName": "Thời Gian Thực (Tạm Dừng Chiến Thuật)",
  "description": "Đánh liên tục trên bản đồ, nhấn dừng để ra lệnh. Liền mạch với thế giới.",
  "driverKey": "core:rtwp",
  "usesGrid": true,
  "available": true
}
// core_turnbased.json
{
  "id": "core:turnbased",
  "displayName": "Theo Lượt (hàng trước / hàng sau)",
  "description": "Ra đòn theo lượt. Đội hình 2 hàng: cận chiến giữ hàng trước, tầm xa nấp hàng sau. Đổi đồng đội, tính khắc hệ.",
  "driverKey": "core:turnbased",
  "usesGrid": false,
  "available": true,
  "config": { "activeSlotsPerSide": 3, "rowsPerSide": 2, "switchCostsTurn": true }
}
```

- **Mode mới đơn thuần đổi tham số / reskin** = chỉ thêm JSON.
- **Mode có luật xếp lịch thật sự mới** = thêm 1 class `ICombatDriver` + 1 dòng `Register`. Đó là ranh
  giới CƠ CHẾ⟂NỘI DUNG áp cho combat mode.
- `available` = chỗ trống cho achievement mở khoá mode về sau (mặc định true).

### 4.4 Driver ĐỘC LẬP — ranh giới chia sẻ nằm ở đâu (chốt 2026-09-02)

> Yêu cầu user: *"nếu tạo driver mới có hiệu quả tốt hơn dùng lại elemental cũ thì đừng gượng ép để tiết
> kiệm. Thứ ta cần là sự independent của driver để hoạt động hoàn hảo nhất, không cần lo về quota."*

**Bắt buộc chia sẻ (mọi driver phải xài chung — đây LÀ trục A):**

| Lớp | Vì sao chung |
|---|---|
| `SkillDefinition` + `Data/skills`, `CombatUnitDefinition` + `Data/combat_units` | 1 skill phải đánh được ở mọi mode (chốt §9.2). |
| **Combat Model**: `Combatant` (bọc `PawnBody`), `Ability`, `EffectPipeline` + registry `EffectWorker`, `AilmentRuntime`, `DamageCalculator`, bảng khắc hệ | Đây là "một skill LÀM GÌ" — độc lập với "khi nào nó resolve". Fork lớp này = fork nội dung = vỡ A5/A6. |
| `DeterministicRng` (từ `CombatRuntime`), `CombatEncounter` (vòng đời + điều kiện thắng), `ICombatSession` (bề mặt UI) | Hạ tầng chung, không mang luật mode. |

**Được phép độc lập hoàn toàn (nội thất của từng driver — KHÔNG ép tái dùng):**

- Vòng thời gian / vòng lượt, initiative, hàng đợi lệnh.
- Di chuyển: RTwP dùng `GridMovementSystem` + A*; turn-based hàng-trước/sau **không cần di chuyển ô** →
  không kéo `GridMovementSystem` vào.
- Cast: RTwP dùng `SkillCastController` (đếm tick); turn-based có thể resolve tức thì trong lượt →
  không cần.
- AI: RTwP dùng `CombatBehaviorSystem` (goal-selector realtime); turn-based cần bộ chọn nước đi theo
  lượt, đơn giản hơn nhiều → **viết mới cho khít** thay vì bẻ cong cái cũ.
- Input người chơi, camera, HUD.

**Cách hoà với luật A6 (cấm fork đường ống):** A6 cấm fork *đường ống thực thể / dữ liệu* (bài học
Disciple⟂PawnBody, Companion fork stack AI). Nó **không** cấm mỗi driver có bộ lập lịch riêng — đó chính
là thứ cái seam `ICombatDriver` được sinh ra để cho phép. Lằn ranh: fork **`SkillDefinition` / Combat
Model / data** = vi phạm A6; nội thất scheduling riêng theo driver = đúng thiết kế.

**Phép thử:** thêm mode thứ 3 (vd auto-battler) → chỉ cần 1 class driver mới + JSON, **không đụng**
`SkillDefinition`, `EffectWorker`, hay 2 driver cũ.

---

## 5. Điểm mấu chốt trục A — `SkillDefinition` phải viết ở mức NGỮ NGHĨA

**Đã chốt: turn-based DÙNG CHUNG `Data/skills` với RTwP.** Không có thư mục move riêng (luật A6 — không
fork data). Nghĩa là 1 `SkillDefinition` phải diễn giải được ở cả hai mode.

`SkillDefinition` hiện tại **bị nướng chặt vào RTwP + lưới**: `castTime`/`cooldown` tính bằng **giây**,
`hitShapeType`/`targetingMode`/`hitShapeRange` là **footprint trên lưới**, `knockbackTier` là **đẩy ô**.
Turn-based (hàng trước/sau) không có toạ độ, không có "giây".

Giải pháp — **thêm field ngữ nghĩa, giữ field cũ, mỗi driver đọc phần của mình** (luật B3: enum/field đã
serialize là APPEND-ONLY nên chỉ được thêm):

| Khái niệm | Field RTwP đọc (đã có) | Field ngữ nghĩa thêm mới | Turn-based đọc |
|---|---|---|---|
| Chọn mục tiêu | `hitShapeType` + `targetingMode` + `hitShapeRange` → footprint | `targetClass` enum: `Self / OneEnemy / AllEnemies / OneAlly / AllAllies / Everyone / RandomEnemy` | `targetClass` |
| Hàng trước/sau | (không dùng — lưới lo vị trí) | `rowRule` enum: `Any / FrontOnly / PrefersFront` (mặc định `Any`) | `rowRule` + `row` của combatant (§6) |
| Nhịp ra đòn | `castTime` (giây), `cooldown` (giây) | `actionCost` (mặc định 1 = khoá bao lượt), `cooldownTurns` (mặc định 0) | `actionCost` + `cooldownTurns` |
| Chi phí | `spiritCost` (Linh Lực) | `charges` (mặc định -1 = vô hạn) ← "PP" | `charges` |
| Thời lượng ailment | ailment "duration" hiểu là giây | `durationTurns` trên `AilmentDefinition` | tick 1 lần / vòng |
| Khắc hệ | (chưa có bảng khắc hệ đúng nghĩa) | `DamageTypeDef.relations`: `strongVs / weakVs / immuneVs` theo id | **dùng CHUNG** — nằm trong `core:damage` worker, thuần data, không dính driver |

**Quy tắc canonical:** `targetClass` là field CHÍNH. RTwP driver có bảng map `hitShapeType → targetClass`
mặc định để **JSON skill cũ chạy tiếp không cần sửa**; `hitShapeType` ở lại như phần tinh chỉnh KHÔNG
GIAN riêng của RTwP. Skill mới author `targetClass`. Skill chỉ-RTwP (không bao giờ xuất hiện ở turn-based)
được phép bỏ trống `targetClass`.

**`rowRule` suy mặc định từ `weaponType`** để không phải author lại 133 skill: `fist/sword/spear/blade/
greatsword/fan/hidden` → `PrefersFront`; `bow` → `Any`; skill `affectsAllies` hoặc `targetClass=Self` →
`Any`. Bảng suy này ở RTwP-map chung, override bằng field tường minh khi cần.

**Nhắm ngữ nghĩa chứ không nhắm footprint** — để mỗi driver tự bung. RTwP bung `targetClass` → footprint
trên lưới; turn-based đọc thẳng. Làm ngược lại (author footprint rồi ép turn-based suy ngược) là sai.

**Override theo mode** (khi thật sự cần "skill này mạnh hơn ở turn-based"): block optional
`modeOverrides: [{ modeId, field, value }]` — cùng khuôn `List<TraitModifierEntry>` dự án đã dùng.
**Chưa xây tới khi vướng thật.**

---

## 6. Turn-based hàng trước / hàng sau — cần gì (Phase 3)

Mô hình FFX / Persona: **phi lưới nhưng có ĐỘI HÌNH 2 hàng mỗi bên.**

- **Slot + hàng:** không `CombatGrid`. Mỗi bên có `activeSlotsPerSide` combatant hoạt động (config, mặc
  định 3), mỗi combatant mang cờ `row: Front | Back`. Băng ghế dự bị = phần còn lại của party (M57).
- **`row` quyết định gì:**
  - `rowRule=FrontOnly` / `PrefersFront` (cận chiến): chỉ đánh được hàng trước ĐỊCH; hàng sau địch bất
    khả xâm **chừng nào hàng trước còn người**. Hàng trước địch sạch → hàng sau tự thành mục tiêu hợp lệ.
  - `rowRule=Any` (tầm xa / phép): đánh hàng nào cũng được.
  - Combatant đứng hàng sau: nhận sát thương cận chiến **giảm** (vd ×0.5) khi bị đánh tới; đổi hàng tốn
    1 action (hoặc auto khi hàng trước cùng đội trống).
- **Initiative:** sắp xếp `(Agility desc, unitId asc)` mỗi vòng — **không bao giờ** theo thứ tự Dictionary
  (luật A4). Skill có tag `"priority"` đi trước bất kể tốc độ.
- **Vòng lặp:** mỗi combatant còn sống được 1 action / vòng (trừ khi `actionCost > 1` khoá qua vòng sau).
  Hết vòng → tick ailment 1 nhịp → kiểm thắng → vòng mới.
- **`knockback` / `hitShape` bị bỏ qua** (driver `UsesGrid=false`). `knockbackTier` cao có thể map thành
  "đẩy địch về hàng sau" — tuỳ chọn, làm sau.
- **Đổi đồng đội:** đọc `PawnPartyRegistry` (M57) làm băng ghế. "Tung người kế" = swap combatant active
  ↔ dự bị. `switchCostsTurn` trong config. RTwP `SupportsPartySwitch=false`.
- **PP:** `charges` trên skill; hết charge → skill xám. Hồi khi rời trận (hoặc item — sau).
- **UI tối thiểu:** menu chọn nước đi, bảng thứ tự lượt, con trỏ chọn mục tiêu (hiện rõ hàng trước/sau),
  nút đổi người. Bind qua `ICombatSession` mở rộng; UI check `Session.UsesGrid` để ẩn phần lưới/camera-pan.

---

## 7. Chỗ đặt lựa chọn mode

### 7.1 Lưu — `SaveData.combatModeId`

```csharp
// SaveData.cs — field mới, append (luật B3)
/// <summary>Lối đánh của ván chơi (M64). CỐ ĐỊNH lúc tạo ván, KHÔNG đổi giữa chừng —
/// cùng hạng với seed/realm: cấu hình THẾ GIỚI, không phải tuỳ chọn phiên.
/// Save cũ (trước M64) = null/rỗng → "core:rtwp" (luật B4: sentinel phân biệt "chưa có").</summary>
public string combatModeId;
```

`SaveData.NewGame(saveName, player)` → thêm tham số `string combatModeId`. Chỗ đọc (khởi tạo
`CombatRuntime`) resolve `DataRegistry<CombatModeDefinition>.TryGet(save.combatModeId ?? "core:rtwp")`.

**Đã chốt:** cố định theo ván. Cho đổi trong Settings **chỉ tính tới ở cuối dự án** — thiết kế chừa cửa
(field là string, không enum; không có logic nào giả định "mode không đổi được") nhưng M64 **không xây**
đường đổi mode, và không migrate trạng thái giữa 2 mode.

### 7.2 Màn chọn — NGAY TRÊN TRANG ĐẦU MENU (đã chốt)

`MainMenuController.MainPanel` (trang đầu, hiện có 3 nút Bắt Đầu / Cài Đặt / Thành Tựu) — thêm **bộ chọn
lối đánh** ngay tại đây, cạnh/trên cụm nút.

- Bộ chọn = hàng card / segmented control **data-driven** từ `DataRegistry<CombatModeDefinition>` (đúng
  `feedback_data_driven_taskbar_panels`: JSON + grid, không hard-code). Card ẩn nếu `available=false`.
- Giá trị đang chọn là **mặc định truyền vào `SaveData.NewGame`** khi người chơi bấm Bắt Đầu → Tạo Mới.
- **Tiếp tục save cũ**: bỏ qua bộ chọn này, dùng `save.combatModeId` của chính save đó. Bộ chọn trên
  MainPanel chỉ áp cho ván **mới tạo**. (Có thể hiển thị mode của save đang chọn ở SaveListPanel cho rõ.)
- Luồng: `MainPanel [chọn mode + Bắt Đầu] → SaveListPanel → [Tạo Mới] → CharacterCreation →
  OnCharacterCreated(playerData) → SaveData.NewGame(name, player, chosenModeId)`.

### 7.3 `CombatSandbox.unity` — môi trường độc lập (dev + chơi + debug)

Scene mới, **không phụ thuộc save / world state**. Vai trò: phát triển toàn bộ M64 tách biệt, ghép vào
Wild ở phase cuối.

- **Màn chọn tướng đơn giản:** 2 danh sách (phe player / phe địch), bốc pawn từ `DiscipleDatabase` +
  `Data/combat_units` (tái dùng đúng lối `_useDiscipleTestRoster` trong `CombatCoreDriver`). Dropdown chọn
  mode — **đọc thẳng từ `CombatDriverRegistry`** (mọi mode đã Register đều hiện). Nút Start → `CombatRuntime`.
- **Boot tối thiểu nhưng THẬT:** vẫn qua `GameRoot`/`ServiceLocator` để có `DataRegistry`, `GlobalEventBus`,
  `GameClock`. Không stub — nếu Combatant dựng được ở đây thì dựng được ở Wild.
- **Là hạt giống của `CombatArena.unity`:** phase 4 lấy chính scene này (thêm backdrop/camera) làm màn đấu
  additive cho `ArenaEncounter`. Không dựng 2 scene.
- **Kỷ luật:** sandbox chạy mọi thứ qua `ICombatDriver` y hệt Wild. Cái gì chỉ chạy trong sandbox = bug.
- `combat_in_prototype.unity` cũ vẫn giữ nguyên cho harness RTwP M22 (phase 0 verify không đổi hành vi).

---

## 8. Các ràng buộc dễ quên

- **Determinism (A4):** driver nhận `DeterministicRng` từ `CombatRuntime`. Turn-based: mọi roll
  (trúng/chí mạng/proc trạng thái) rút từ CÙNG stream; thứ tự initiative sort ổn định.
- **Save giữa trận:** NGOÀI phạm vi M64 v1 — combat là encounter nhất thời. Nếu turn-based về sau đủ dài
  cần lưu giữa trận: trạng thái trận (số vòng, con trỏ initiative, tầng chỉ số tạm, bộ đếm lượt ailment)
  thành 1 blob serializable **do driver sở hữu**, dựng khi vào trận, xé khi `Ended`. Kết quả BỀN (HP qua
  M30, chết, loot, XP) vẫn commit qua hệ sẵn có lúc `Ended`. Ghi ra đây, chưa xây.
- **Luật A5 — phép thử:** xoá `Sim/Xianxia/` → **cả hai mode vẫn giải quyết trận đủ**, chỉ mất cơ chế Khí
  Thuẫn / Linh Lực (chúng là `EffectWorker` của xianxia-pack). Mode "turn-based kiểu Pokémon" nằm ở
  core-pack là hợp lệ — nó là cơ chế, không phải nội dung tu tiên.
- **`ICombatSession` mở rộng, không thay:** vài thành viên hiện tại thiên RTwP (`GridVisualRef`, slow-mo,
  `AutoPauseOnReady`). Turn-based trả `null` / no-op; UI nhánh theo `UsesGrid`. Không đập interface.

---

## 9. ĐÃ CHỐT (2026-09-02)

1. **Bố cục turn-based:** hàng trước / hàng sau (FFX, Persona) — phi lưới, đội hình 2 hàng. Xem §6.
2. **Nguồn skill:** dùng CHUNG `Data/skills` + thêm field ngữ nghĩa (§5). Không thư mục move riêng.
3. **Tính bền của mode:** cố định theo ván. Đổi trong Settings để tới cuối dự án; M64 chừa cửa nhưng
   không xây (§7.1).
4. **Màn chọn:** bộ chọn data-driven ngay trên `MainMenuController.MainPanel` — trang đầu menu (§7.2).
5. **Driver ĐỘC LẬP tuyệt đối:** xem §4.4. Mỗi driver được tự viết nội thất riêng (vòng lượt, initiative,
   di chuyển, AI, targeting). KHÔNG ép driver mới xài lại linh kiện của driver cũ chỉ để tiết kiệm — chỉ
   tái dùng khi linh kiện đó thật sự tốt. Cái phải chia sẻ là **Combat Model + data**, không phải nội
   thất RTwP.
6. **Xây trong scene độc lập trước:** `CombatSandbox.unity` + màn chọn tướng đơn giản (§7.3). Phát triển
   + chơi + debug tách biệt, mọi thứ vẫn qua `ICombatDriver`/`CombatDriverRegistry`. Nối vào Wild ở phase
   4 (cuối). Scene này tiến hoá thành `CombatArena.unity`.

---

## 10. Lộ trình — **xây trong scene độc lập trước, ghép vào Wild sau cùng** (chốt 2026-09-02)

> Chiến lược: dựng `CombatSandbox.unity` làm môi trường độc lập vừa test vừa chơi vừa debug. Mọi thứ đi
> qua `ICombatDriver` + `CombatDriverRegistry` + Combat Model y như đường thật. Hệ ổn rồi mới nối dây vào
> Wild — nối cuối cùng, một lần. Scene sandbox này chính là hạt giống của `CombatArena.unity` (§7.3).

| Phase | Nội dung | Verify | Trạng thái |
|---|---|---|---|
| **0** | Seam `ICombatDriver` + `CombatDriverRegistry` + `CombatSetup` + `RtwpCombatDriver` (bọc `CombatRuntime` nguyên trạng, **không viết lại**). `Combat/Drivers/*`. | Test `Run M64 CombatDriver Tests` (18/18) + Play Mode qua sandbox. M22 (`CombatCoreDriver` + `WildCombatSystem`) **0 dòng thay đổi** ⇒ không đổi hành vi theo cấu trúc. | ✅ **XONG 2026-09-02** |
| **1** | `CombatSandbox.unity` + `CombatSandboxController` (OnGUI: chọn tướng từ `Data/combat_units`, dropdown mode từ `CombatDriverRegistry`, Bắt đầu/Dừng/Dừng hình, bảng theo dõi HP/vị trí/nạp/ailment; `playerTeam=None` ⇒ AI tự đánh cả 2 phe để xem trận). `Combat/Sandbox/*`. | ✅ Play Mode 2026-09-02: sandbox auto-start `[core:rtwp] 1v1`, sói vượt map (13,1)→(3,1) đánh nhau, sói chết (HP 0/120), player 125/240, `Ended` bắn `winner=A`; Pause/Resume/Toggle/Restart qua seam đều đúng; **0 error/warning** cả phiên. | ✅ **XONG 2026-09-02** |
| **2** | `TurnBasedCombatDriver` **greenfield, plain C#** (`Combat/Drivers/TurnBased/`): `TurnCombatant` + `TurnStatus` + `TurnSkillView` (đọc `SkillDefinition` ở mức ngữ nghĩa, SUY từ field sẵn có — chưa thêm field mới) + driver (round loop, initiative theo SPD, target resolver hàng-trước/sau, DoT/stun tick/vòng, cận chiến ×0.5 hàng sau, `Tick` tự advance). `DamageCalculator` thêm overload `CharacterStats` (Combat Model dùng chung — RTwP overload uỷ thác, hành vi giữ Y HỆT — xem §10.2). Đăng ký `core:turnbased`. Sandbox `Update()` pump `Tick` + bảng theo dõi. | ✅ `Run M64 TurnBased Tests` **24/24** headless (trận chạy tới KẾT THÚC, tất định, pause). ✅ Play Mode 2026-09-03: sandbox `core:turnbased` 2v2, đội hình Front/Back đúng, sói (Front) chết trước rồi hươu (Back), `winner=A` sau 4 vòng, `Tick` auto-advance. `Run M64 CombatDriver Tests` **19/19**. | ✅ **XONG 2026-09-03** |
| **3** | Data + save (như trên) **+ bộ chọn mode trên `MainPanel`**: `CombatModeDefinition.shortLabel`/`ButtonLabel` + `UI/MainMenu/CombatModeSelector.cs` (data-driven, clone nút mẫu từ `StartButton` — không hard-code số mode) + `MainMenuController._combatModeSelector` luồn `SelectedModeId` vào `SaveData.NewGame`. Continue-save dùng `combatModeId` của chính save. | ✅ Play Mode 2026-09-03: MainPanel hiện 2 nút "Thời Gian Thực" / "Theo Lượt" từ `Data/combat_modes/`; `Select("core:turnbased")` → `NewGame` ra `combatModeId="core:turnbased"`; `NewGame(null)` → `"core:rtwp"`. | ✅ **XONG 2026-09-03** |
| **4** | **Nối dây vào Wild** — chốt chặn `WildCombatOps.ArenaInterceptor` (static Func, 1 dòng guard trong `ResolveAttack`) + `World/Wild/Combat/ArenaCombatBridge.cs` (tự gắn qua `LocalMapBootstrap.EnsureMapSystems`, tự TẮT nếu mode ≠ `core:turnbased` ⇒ seamless M22 0 thay đổi). Khi bật: đòn pawn⟂quái đầu tiên → đóng băng `GameClock` + dựng 2 `TurnCombatant` từ `IWildCombatant.Carrier.Stats` THẬT → `TurnBasedCombatDriver.BeginWith` → OnGUI HUD → `Ended` ghi chênh lệch máu/chết/loot về thực thể gốc → thả pause. | ✅ Play Mode 2026-09-03 (sandbox + thực thể thật): `ResolveAttack` trả `Executed=False` (nuốt), HP gốc đứng yên trong lúc đánh, arena 4 vòng `winner=A`, ghi về: pawn 220→132.5, sói 120→0 `alive=False`, `_clockPause` thả. `Run WildCombat Tests` 53/53. | 🔶 **cơ chế XONG 2026-09-03; giới hạn ở §10.3** |

Mỗi phase verify Play Mode trước khi sang phase sau.

### 10.1 Ghi chú thực thi Phase 0-1 (2026-09-02)

- **"Combat Model" CHƯA tách thật ở Phase 0.** Doc bản đầu ghi Phase 0 tách EffectPipeline/EffectWorker
  — nhưng thứ đó **chưa tồn tại**, là dựng mới chứ không phải extract, và là refactor lớn đụng
  damage/ailment/skill khắp nơi ⇒ hoãn sang khi Phase 2 (turn-based) thật sự cần resolve skill theo
  cách riêng. Phase 0 tối thiểu = seam + registry + `RtwpCombatDriver` bọc `CombatRuntime`. Model chung
  hiện dùng tạm: `CombatEncounter` (điều kiện thắng) + `CombatUnit` + `SkillDefinition` +
  `DamageCalculator` + `DeterministicRng` — đều đã tái dùng được độc lập.
- **Hai stack combat cũ, chỉ đụng cái RTwP:** `CombatRuntime`+RTwP (prototype, chỉ vào qua
  `combat_in_prototype.unity`) LÀ cái Phase 0 bọc. `WildCombatSystem` (seamless static-ops, gameplay
  Wild thật của M22) **không đụng tới** — Phase 4 mới quyết quan hệ mode ⟂ seamless.
- **Edit Mode test hẹp có chủ ý:** RTwP chạy trên vòng đời MonoBehaviour (`CombatEncounter.Awake`) nên
  KHÔNG dựng được headless ngoài Play Mode ⇒ test menu chỉ phủ registry/resolve/cờ/guard; hành vi trận
  verify ở sandbox Play Mode.
- **`CombatSandbox.unity` chưa vào Build Settings** — scene dev, mở từ Project window. Vào Build Settings
  (hoặc hoá thân `CombatArena.unity`) ở Phase 4.
- **Chưa Play-test lại `combat_in_prototype.unity`** — 0 dòng thay đổi ở file của nó + compile sạch nên
  an toàn theo cấu trúc; nếu muốn chắc tuyệt đối thì mở scene đó bấm Play 1 lần.

### 10.2 Ghi chú thực thi Phase 2-3 — viết code-only 2026-09-02, verify 2026-09-03

Viết trọn khi Unity tắt (không MCP). Mở lại Unity 2026-09-03: **1 lỗi compile duy nhất** — `CS0102`
`TurnBasedCombatDriver` trùng tên `Ended` (event của interface ⟂ property introspection). Đổi property
thành `IsEnded`. Sau đó:

- ✅ `Run M64 CombatDriver Tests` **19/19** · `Run M64 TurnBased Tests` **24/24** (headless: trận turn-based
  chạy tới KẾT THÚC không treo, tất định theo seed, cổng Pause + AutoAdvance, smoke overload DamageCalculator).
- ✅ Play Mode 2026-09-03: `CombatSandbox.unity` chạy `core:turnbased` 2v2 (player_placeholder+swordsman
  vs wolf+deer). Đội hình Front/Back gán đúng. Sói (Front) chết trước, rồi hươu (Back) → `winner=A` sau
  **4 vòng**, `Tick` từ `Update()` tự advance. `Ended` bắn đúng.
- ✅ Hồi quy: `Run WildCombat Tests` **30/30**, `Run M Combat Skill Tests` **7/7**.

⚠ **`DamageCalculator` refactor — CHỨNG MINH KHÔNG hồi quy, nhưng lộ 4 test ĐỎ CÓ SẴN TỪ TRƯỚC:**
`Run M3 Tests` (DamageCalculator ca: expect 50, got **40**) + `Run M Combat Core Tests` (3 đỏ: AOE
`192`/`240`, `bleed không áp được`). **Không phải do M64.** Chứng minh: overload `CombatUnit` cũ dời body
**verbatim** + tính `victimExposed` bằng ĐÚNG dòng gốc `victim.Shields != null ? victim.Shields.IsExposed
: !victim.Health.HasShield`; chạy `execute_code` live: overload `CombatUnit` trả **40** (khớp hành vi
gốc), overload mới `victimExposed:true`→50 / `false`→40. Không call-site cũ nào dùng overload mới.
- **Bug tiềm ẩn có sẵn (ngoài phạm vi M64, KHÔNG sửa ở đây):** `CombatUnit.Shields` **luôn ≠ null** (tạo
  trong ctor) ⇒ nhánh fallback `!Health.HasShield` là **code chết**; `QiShieldController.IsExposed` mặc
  định `false` cho unit chưa từng có thuẫn ⇒ **mọi unit KHÔNG thuẫn bị ăn nhầm mitigation ×0.8** ("còn
  thuẫn → 80%"). Test M3/MCombatCore viết `expected` từ trước khi có nhánh 0.8. Sửa = đổi số damage RTwP
  cho mọi unit không thuẫn (thay đổi cân bằng) — việc riêng, đã spawn task.

**Giới hạn Phase 2 đã biết (không phải bug):** (a) đổi party = có cơ chế (`SupportsPartySwitch=true`)
nhưng CHƯA nối `PawnPartyRegistry` (cần `PawnRef`, làm ở Phase 4); băng ghế hiện rỗng. (b) PP vô hạn —
`TurnSkillView.MaxCharges` trả 0; field `charges` thật + `targetClass`/`rowRule`/`actionCost` trên
`SkillDefinition` là việc còn lại của Phase 3 (append field, sửa các hàm Derive* trong `TurnSkillView`).
(c) chưa có điều khiển người chơi — AI chạy cả 2 phe. (d) initiative dùng `SPD` (chỉ số combat suy từ
Agility) chứ không phải `Agility` thô như §6 ghi — `SPD` khớp với cách RTwP xếp nhịp, cập nhật §6 sau.

### 10.3 Ghi chú Phase 3 UI + Phase 4 (2026-09-03)

**Phase 3 UI — XONG.** `CombatModeSelector` (`UI/MainMenu/`) clone nút mẫu (`ModeButtonTemplate`, bản sao
`StartButton` bị ẩn) 1 nút / mode `available`, đọc nhãn từ `CombatModeDefinition.ButtonLabel`
(`shortLabel` → fallback `displayName`). Dựng trong `MainMenu.unity` dưới `ButtonGroup` (sibling 0), wire
`MainMenuController._combatModeSelector`. `OnCharacterCreated` đọc `SelectedModeId` → `SaveData.NewGame`.
Continue-save KHÔNG đụng bộ chọn (dùng `save.combatModeId`). Nhãn "Thời Gian Thực" xuống 3 dòng vì nút
hẹp — cosmetic, sửa bằng cách đổi `shortLabel` trong JSON hoặc nới nút.

**Phase 4 — CƠ CHẾ nối Wild XONG, còn giới hạn:**
- `WildCombatOps.ArenaInterceptor` (static `Func<IWildCombatant,IWildCombatant,bool>`, mặc định null) +
  1 dòng guard trong `ResolveAttack` (sau check InRange). Null ⇒ seamless M22 y nguyên (`Run WildCombat
  Tests` 53/53 xác nhận).
- `ArenaCombatBridge` (`World/Wild/Combat/`, tự gắn `EnsureMapSystems`) đọc `SaveManager.ActiveSave.
  combatModeId`; ≠ `core:turnbased` → `enabled=false`. Bật thì set interceptor; đòn pawn⟂quái đầu tiên →
  `PushPause(CombatPrototype)` + dựng 2 `TurnCombatant` từ `wc.Carrier.Stats` + `wc.CurrentHP/MaxHP` +
  `Source=wc` → `TurnBasedCombatDriver.BeginWith(...)` → OnGUI HUD → `Ended`: ghi `delta = src.CurrentHP -
  c.Hp` về `src.ApplyDamage/Heal`, quái chết → `WildCombatOps.DropLoot`, thả pause.
- `TurnBasedCombatDriver.BeginWith(IEnumerable<TurnCombatant>, Team, seed)` — nhánh Begin không qua
  `CombatSetup`/`UnitFactory`. `TurnCombatant.Def` giờ **được phép null** (đấu sĩ Wild), `elementAffinity`
  fallback 0. `TurnCombatant.Source` mới.

**GIỚI HẠN Phase 4 (nói thẳng, chưa làm):**
1. **Arena 1v1** — chỉ attacker + target, chưa gom pawn/quái lân cận vào cùng trận (cần quét scene theo
   bán kính). Hàng-trước/sau vì thế chưa phát huy (1 người/phe → đều Front).
2. **Chỉ đánh thường** — `IWildCombatant` không lộ bộ skill; đấu sĩ arena Wild chưa dùng skill của
   pawn/quái. Nối = map `Disciple.Books`/`CombatUnitDefinition.skills` → `TurnCombatant.Skills`.
3. **HUD OnGUI** — chưa có `CombatArena.unity` scene riêng / Canvas. Doc §7.3 nói sandbox tiến hoá thành
   scene này; hiện vẫn overlay.
4. **Chưa nối `PawnPartyRegistry`** làm băng ghế (SupportsPartySwitch=true nhưng bench rỗng).
5. **Seed arena hardcode** `12345` trong `BeginWith` — nên suy từ `save + encounter id` để replay ổn định.
6. **XP / M30 injury** — write-back hiện chỉ HP (qua `ApplyDamage` → `WildInjuryOps.OnDamageApplied`
   KHÔNG chạy vì bỏ qua `ResolveAttack`). Vết thương M30 + XP chưa commit từ arena.
7. **Chưa Play-test trên `WildLocal.unity` thật** — verify 2026-09-03 chạy trong `CombatSandbox` với
   `PawnCombatant`/`BeastCombatant` dựng từ `Disciple`/`BeastActor` THẬT + `SaveManager.ActiveSave` ép
   `core:turnbased`. Đường bootstrap Wild (party spawner, avatar) chưa chạy qua.

**Phần Phase 3 gốc còn lại (đã làm xong ở trên):** ~~bộ chọn mode trên MainPanel~~. `.meta` cho
`Data/combat_modes/*.json` Unity tự sinh khi import (đã có).

### 10.4 Phase 5 — Arena kiểu Pokémon hiện đại + hệ skill + Canvas UI (2026-09-03)

Giải quyết nợ Phase 4 mục (2) skill, (3) HUD OnGUI, (4) PawnPartyRegistry, và làm rõ (1) "arena 1v1".
Chốt: **lối đánh `core:turnbased` = kiểu Pokémon Sword/Shield** (user nghiêng "hiện đại") — 1 đấu sĩ mỗi
bên ra trận + băng ghế dự bị, đổi đấu sĩ là 1 hành động. Cơ chế hàng-trước/sau của Phase 2 **vẫn còn**
trong driver (đường `ActivePerSide == 0`) cho mode FFX tương lai / test headless.

**Driver — mô hình CHỐT-LỆNH (`TurnBasedCombatDriver` viết lại):**
- `int ActivePerSide` — `0` = MỌI đấu sĩ ra tay mỗi vòng (FFX, mặc định, 24 test Phase 2 giữ nguyên);
  `≥1` = Pokémon: N con đầu-tiên-còn-sống mỗi phe ra trận, còn lại băng ghế.
- Mỗi vòng: `StartRound()` tính nhóm ra trận → AI chốt lệnh NGAY (`PickAiCommand`), đấu sĩ
  `TurnCombatant.PlayerControlled` vào `PendingCommand` chờ. `SubmitCommand(actor, TurnCommand)` — đủ
  lệnh thì `ResolveRound()` chạy ngay: sắp theo SPD, `ExecuteCommand` từng đấu sĩ (tái dùng
  `ResolveTargets`/`ResolveHit` Phase 2), tick status/cooldown cuối vòng, `CheckWin`.
- `TurnCommand` (struct) — `Attack` / `Skill(i)` / `Switch(benchIndex)` / `Flee` / `Skip`.
- `Tick()` **KHÔNG tự advance khi `AwaitingCommands`** — dừng chờ người chơi. `AdvanceRound()` GIỮ LẠI
  làm đường "AI cho tất cả" (tự điền pending) → 24 test Phase 2 chạy y nguyên với `ActivePerSide=0`.
- `ActiveOf(Team)` / `BenchOf(Team)` cho UI. `SwitchIn` hoán vị trong `_combatants` (con băng ghế lên
  đầu). Đấu sĩ ngã → `ComputeActingSet` tự lấy con kế còn sống vòng sau (không có prompt bắt đổi — nợ).
- `TurnSkillView.MaxCharges` giờ suy PP từ `skillTier` (weak 20 / medium 15 / strong·onhit 10 / 15) —
  `TurnCombatant.SeedCharges` + `PpOf(sk)` → "x/y" cho UI. Field `charges` thật trên `SkillDefinition`
  vẫn là nợ (mục 8 §10.3).
- `AddSkill` giờ `public static` (arena + scene controller nạp bộ chiêu vào đấu sĩ), lọc bỏ
  passive/normal/summon, trần 4 chiêu (kiểu Pokémon).

**`Data/combat_modes/core_turnbased.json` (×2 thư mục):** `activeSlotsPerSide: 3→1`, `rowsPerSide: 2→1`,
displayName/description đổi sang "kiểu Pokémon". `core_rtwp.json` giữ nguyên (`activeSlotsPerSide: 0`).

**UI — `ArenaBattleUI` (`World/Wild/Combat/`, thay OnGUI):** Canvas dựng HOÀN TOÀN bằng code
(`RenderMode.ScreenSpaceOverlay`, `sortingOrder 30000`, `LiberationSans`/legacy `Text` — né bẫy
[[project_hud_prefab_unify]] + bẫy nạp scene bất đồng bộ giữa trận). Bố cục SwSh: name-plate địch
(trên-trái) + phe ta (dưới-phải) với thanh HP lerp; hộp thoại "{tên} sẽ làm gì?"; menu ĐÁNH/TÚI/ĐỔI/CHẠY;
ĐÁNH → tối đa 4 nút chiêu (tên + `<hệ>` + PP x/y, mờ nếu chưa hồi); ĐỔI → danh sách băng ghế. Tự bơm
`driver.Tick(Time.unscaledDeltaTime)` (arena chạy độc lập khi GameClock bị đóng băng). Chưa có bảng khắc
hệ ⇒ **chưa hiện nhãn "hiệu quả"** (nợ, mục dưới). TÚI vô hiệu (arena chưa có hệ item).

**`ArenaCombatBridge` (Phase 4→5):** thay OnGUI bằng `ArenaBattleUI`. `Make()` nạp skill thật
(`BeastCombatant` → `CombatUnitDefinition.normalAttackId`/`activeSkill1Id`/`activeSkill2Id`/`skills`;
`PawnCombatant` → `PawnBody.Skills[].Definition`), set `PlayerControlled = team==A`, `Level` proxy theo
MaxHp. `ActivePerSide` đọc từ `CombatModeDefinition.config.activeSlotsPerSide`. **Băng ghế:**
`AddPlayerBench` gom `PawnPartyRegistry.PartyOf(pawn.Ref).memberRefs` → strip `"disciple:"` →
`DiscipleRegistry.Get()` → `TurnCombatant` (Team.A, `PlayerControlled`, `Source = PawnBody`). Phe địch
GIỮ 1v1 (đúng "chạm quái hoang" kiểu Pokémon). Write-back `OnArenaEnded` xử thêm `c.Source is PawnBody`
(đồng đội băng ghế không ở trên map) qua `body.ModifyHP`.

**`CombatArena.unity` + `ArenaSceneController` (`World/Wild/Combat/`):** scene phụ đứng-một-mình để
xem/tinh chỉnh UI — dựng trận từ `Data/combat_units` id rồi mở `ArenaBattleUI`. Đường Wild THẬT KHÔNG
nạp scene này (bridge dựng UI thẳng bằng code — không có pha nạp bất đồng bộ giữa trận).
✅ **Đã tạo** `Assets/Scenes/CombatArena.unity` (Main Camera + `ArenaBootstrap`/`ArenaSceneController`).
KHÔNG add vào Build Settings (chỉ mở trong Editor để test).

**Test:** `Run M64 TurnBased Tests` **40/40** (24 cũ + 16 Phase 5: ActivePerSide 1⟂0, bench, PP seed/tụt,
Tick-chờ-lệnh-không-advance, Submit/Switch/Flee, AdvanceRound vẫn AI-hoá PlayerControlled). Hồi quy:
`Run M64 CombatDriver Tests` 19/19, `Run WildCombat Tests` 53/53. Compile sạch.

**✅ ĐÃ PLAY-TEST 2026-09-03** (`CombatArena.unity`, Play Mode thật, kiếm sĩ + cung thủ băng ghế vs Lang
yêu): Canvas dựng đúng (sortingOrder 30000, 8 panel), plate địch/ta + HP bar + menu ĐÁNH/TÚI/ĐỔI/CHẠY
hiện đúng; 4 nút chiêu kèm PP (Chém Ngang 15/15 / Đạp Trước 20/20 weak-tier / Đâm Mạnh 15 / Đập Kiếm 10);
`SubmitCommand(Skill 0)` → foe 120→78, PP 15→14, log "Kiếm sĩ Hỏa dùng Chém Ngang! … 42"; `Switch(0)` →
"Kiếm sĩ Hỏa lui về — Cung thủ Băng ra trận!", `ActiveOf(A)` đổi; chơi hết 6 vòng → winner A, Lang yêu
DEAD, màn Result "THẮNG!" + nút "Đóng" → callback `_onClosed` chạy ("Đóng arena (standalone)"). 0 error.
⚠ Bẫy đã biết (M36): Editor mất focus trong phiên MCP thì player-loop `Update()` không tự tick / `Destroy`
hoãn — phải bơm `drv.Tick()` bằng tay khi verify. Lúc user chơi tay (window focus) thì chạy bình thường.

**CÒN NỢ sau Phase 5:**
1. Bảng khắc hệ (type chart) → nhãn "hiệu quả" trên nút chiêu (screenshot user có).
2. Prompt BẮT đổi khi đấu sĩ ra trận ngã (hiện tự lấy con kế).
3. Phe địch nhiều con (giờ luôn 1v1) — nếu muốn "trainer battle".
4. XP + M30 injury commit từ arena (thừa hưởng nợ Phase 4 mục 6).
5. Seed arena vẫn hằng số (nợ Phase 4 mục 5).
6. ~~`SkillDefinition` field ngữ nghĩa~~ ✅ §15 (`actionCost` cố ý để stub — turn-based luôn 1 lượt/đòn).
7. UI `_message.text` gán mỗi frame (concat) — GC nhẹ, chấp nhận cho prototype; gom lại nếu cần.

---

## 12. MODDING — thêm một LỐI ĐÁNH mới (2026-09-03)

Ba tầng, từ "thuần data" tới "C# thật":

### Tầng 1 — BIẾN THỂ config (thuần JSON, không code)

Nếu "lối đánh mới" chỉ là một driver **đã đăng ký** chạy với tham số khác → thả 1 file
`Data/combat_modes/*.json` (hoặc `StreamingAssets/Mods/<Mod>/Data/combat_modes/`):

```json
{
  "id": "mymod:doubles",
  "displayName": "Theo Lượt — Đánh Đôi",
  "shortLabel": "Đánh Đôi",
  "driverKey": "core:turnbased",
  "usesGrid": false,
  "available": true,
  "config": { "activeSlotsPerSide": 2, "rowsPerSide": 1, "switchCostsTurn": true }
}
```

`driverKey` trỏ tới driver lõi; `config` là các knob driver đọc. Mỗi lần dời thêm một luật vào `config`
(initiative, action cost, thứ tự phase…) là thêm một lối đánh nữa mà tầng này làm được.

### Tầng 2 — DRIVER mới (hành vi C#) qua `ICombatModeContributor`

Driver là vòng lặp xếp lịch → phải là code. Modder viết **2 class**:

```csharp
// 1) luật xếp lịch riêng
public sealed class CardBattleCombatDriver : ICombatDriver { /* ... */ }

// 2) contributor — GameRoot tự tìm & gọi lúc boot
public sealed class MyCombatModes : ICombatModeContributor
{
    public void RegisterModes()
        => CombatDriverRegistry.Register("mymod:cardbattle", () => new CardBattleCombatDriver());
}
```

rồi thả `.cs` vào `Assets/` (hoặc ship DLL trong `Assets/Plugins`). `GameRoot` mục **6i** gọi
`CombatDriverRegistry.InstallContributors()` → quét **mọi assembly đã nạp có tham chiếu tới assembly
`ICombatDriver`** (bỏ qua `*-Editor` — driver phải chạy trong build), dựng từng contributor bằng
**constructor rỗng**, gọi `RegisterModes()`. Một contributor ném exception ⇒ **chỉ lối đánh của nó bị
bỏ**, phần còn lại vẫn đăng ký (mirror `ModLoader`). Driver LÕI (`core:rtwp`/`core:turnbased`) vẫn do
`EnsureDefaults()` tự đăng ký — KHÔNG gộp vào scan (test cách ly cần tập lõi xác định).

Sau khi đăng ký, thêm 1 `CombatModeDefinition` JSON (tầng 1) trỏ `driverKey: "mymod:cardbattle"` để lối
đánh hiện trong bộ chọn lúc tạo ván.

### Tầng 3 — không hỗ trợ

Không có đường "tả trọn luồng combat bằng JSON" — cần một ngôn ngữ script, ngoài phạm vi. Tầng 1 mở rộng
dần theo số knob trong `config`.

**API:** `CombatDriverRegistry.Register / InstallContributor(one) / InstallContributors(scan) /
ResetForTests`. Test: `Run M64 CombatDriver Tests` (25/25 — gồm contributor good/throwing/null + scan
idempotent).

---

## 13. Phase 5b — đổi tên driver + khắc hệ + hệ cấp độ (2026-09-03)

### Đổi tên `TurnBasedCombatDriver` → `CreatureDuelCombatDriver`
File `Combat/Drivers/TurnBased/CreatureDuelCombatDriver.cs` (giữ folder + namespace `.TurnBased` —
"turn-based" là từ mô tả, không phải nhãn hiệu). `ModeId` GIỮ `"core:turnbased"` (save cũ). Đã **quét
sạch mọi tham chiếu thể loại game bắt-thú của Nintendo** khỏi code + JSON của M64 (tham chiếu thể loại
CHỈ ở doc này + memory — không đưa vào `Assets/`). Sibling `TurnCombatant`/`TurnCommand`/`TurnSkillView`
giữ tên (generic). *Còn `BeastSpeciesDefinition`/`EventPoolDefinition`/`M_MONSTER_DESIGN.md` có "kiểu
Pokemon" từ trước M64 — ngoài phạm vi, chưa đụng.*

### Khắc hệ (elemental type chart)
- `Data/combat/element_chart.json` (×2) — DATA: mỗi hệ tấn công liệt `strongVs`/`weakVs`/`immuneVs`;
  hệ số ở 3 con số chung (`strongMultiplier` 1.6 / `weakMultiplier` 0.6 / `immuneMultiplier` 0). Ngũ hành
  tương khắc + Phong/Lôi/Băng. Cặp không khai = ×1. Mod sửa/thêm = 1 file JSON.
- `Combat/Damage/ElementChart.cs` — `ElementChartConfig` (Load/Sanitize) + static `ElementChart`
  (`Multiplier(atk,def)` / `Effect(mult)` → `ElementEffect{Immune,Weak,Neutral,Strong}` /
  `EffectLabelVi`). Tên hệ khớp `DamageType` viết thường; int `elementAffinity` map qua
  `DamageCalculator.ElementName`.
- **Chỉ `CreatureDuelCombatDriver.ResolveHit` dùng** — gấp `typeMult` vào `damageMultiplier` TRƯỚC
  `DamageCalculator`. M22/RTwP KHÔNG đụng (khắc hệ là quyết định driver, không phải Combat Model chung).
  `driver.LastEffect` cho UI; log trận ghi `⟪rất hiệu quả⟫`.
- UI: nút chiêu hiện `▲ hiệu quả` / `▼ kém` / `× vô hiệu` so với con địch đang ra trận; sau đòn flash
  "Rất hiệu quả!".

### Hệ cấp độ sinh vật (full + lưu)
- `Data/combat/creature_level_config.json` (×2) — `maxLevel` 100, XP curve
  `round(xpBase·level^xpExponent)`, `statGrowthPerLevel` 0.06 (Lv1 ×1.0 → Lv100 ×6.94), XP thưởng
  `xpFlatPerFoe + xpPerFoeLevel·capDich`, `shareXpAcrossWinners`.
- `Combat/Progression/CreatureLevel.cs` — static thuần tính: `StatMultiplier` / `XpForLevelUp` /
  `TotalXpToReach` / `LevelForTotalXp` (một nguồn sự thật: cấp SUY từ tổng XP) / `XpReward`.
- `Combat/Progression/CreatureLevelRegistry.cs` — sổ `{level, xp}` theo id (quy ước `HealthRegistry`:
  `disciple:{id}` / `beast:{instanceId}`). `LevelOf` (chưa có = 1) / `Ensure(id, minLevel)` (seed, không
  hạ) / `AddXp` → `(newLevel, gained)` / `Entries`/`ReplaceAll`. Đăng ký ServiceLocator ở `GameRoot`
  **6i-bis**, sống xuyên scene như `HealthRegistry`/`MoodRegistry`.
- **LƯU:** `SaveSystem/CreatureLevelIO.cs` (mirror `ChronicleIO`) + `SaveData.creatureLevels`; cắm
  `SaveManager.Load`/`Create`/`WriteFile`. Save cũ = rỗng → mọi con về cấp 1.
- `Combat/Progression/CreatureDuelRewards.GrantXp(driver, winnerTeam)` — cộng XP cho phe thắng còn sống
  (có `SourceId`) từ con bị hạ; trả chuỗi "X lên cấp N!". Bridge + scene controller gọi ở `Ended`.
- **Áp vào trận:** `TurnCombatant += SourceId, LevelMult`. Builder (`ArenaCombatBridge.ApplyLevel` /
  `ArenaSceneController`) đọc registry (seed lần đầu theo máu gốc), gán `Level`/`LevelMult`, SCALE
  `MaxHp`/`Hp` theo cấp (1 chỗ). Driver: `Speed × LevelMult`; `ResolveHit` gấp
  `actor.LevelMult / tgt.LevelMult` vào sát thương (cùng cấp ⇒ ×1). Write-back `OnArenaEnded` đổi sang
  **PROPORTIONAL** ("còn X% máu arena → đặt thực thể về X% máu thật") vì máu arena đã scale.
- Test `Run M64 TurnBased Tests` **64/64** (+15: chart 9, level 15). Hồi quy: `M64 CombatDriver` 25/25,
  `M53 Memory Save` 34/34.

### Forced switch — bắt chọn con thay khi đấu sĩ ở slot ngã
`Phase` thêm `ForcedSwitch`. Cuối `ResolveRound` → `EndOfRoundSwitches()`: con Ở SLOT (N đầu của phe theo
`_combatants`) mà đã ngã + phe còn dự bị sống → nếu `PlayerControlled` **xếp hàng** `_faintedNeedingSwitch`
+ vào `Phase.ForcedSwitch` (return TRƯỚC `CheckWin`); nếu AI thì `SwitchIn` con kế NGAY (y hệt trước, 0
hồi quy). API: `AwaitingForcedSwitch` / `ForcedSwitchFainted` / `ForcedSwitchTeam` / `ForcedSwitchReserves()`
/ `SubmitForcedSwitch(reserveIndex)`. `Tick` đứng im khi `ForcedSwitch`; `AdvanceRound` (đường AI-làm-hết)
tự drain bằng `SubmitForcedSwitch(0)`. UI: `ArenaBattleUI._forcedSwitchMode` — panel ĐỔI hiện
`ForcedSwitchReserves()`, **KHÔNG có "Quay lại"** (phải chọn), message "{con} ngã xuống! Chọn đồng đội ra
thay." Doubles: xếp hàng, xử từng con. Test +13 (`FSwitch_*` — vào trạng thái, Tick không vượt, submit đổi
active, AdvanceRound không treo, AI tự đôn không prompt). **`Run M64 TurnBased Tests` 77/77.**

### 2v2 / 3v3 — nhiều con mỗi phe
Driver vốn đã hỗ trợ `ActivePerSide=N`; đợt này bù các mảnh thiếu:
- **`TurnCommand.Target`** (nullable `TurnCombatant`) — người chơi CHỌN mục tiêu. `Attack(target)` /
  `Skill(i, target)`. `ExecuteCommand` chuyền xuống `DoSkill`/`DoBasicAttack` → `ResolveTargets(..., preferred)`:
  mục tiêu chọn thắng nếu là địch hợp lệ + đang ra trận + chiêu KHÔNG phải AoE / không nhắm đồng minh.
- **`ActivesOf(Team)`** — N con đầu-còn-sống của phe (cho UI nhiều plate + panel chọn mục tiêu).
- **Luật hàng-trước/sau CHỈ áp `ActivePerSide <= 0`** (FFX) — kiểu N-con thì mọi con ngang hàng (nếu không
  AoE `PrefersFront` chỉ trúng 1). Guard cả filter mục tiêu LẪN giảm ×0.5 hàng sau.
- **UI `ArenaBattleUI`** viết lại vòng chốt-lệnh: `_currentActor` = đấu sĩ phe ta đang được ra lệnh; lặp
  qua `PendingCommand` từng con. `Panel.Target` mới — 1 nút / địch đang ra trận (kèm ▲/▼/× khắc hệ), chỉ
  hiện khi đòn đơn-mục-tiêu + >1 địch. Plate: con đầu mỗi phe + dòng phụ `_enemyExtra`/`_playerExtra`
  liệt "+ Tên HP%" cho các con còn lại. Bridge `AddNearbyEnemies` gom ≤ N-1 yêu thú sống Chebyshev ≤ 6
  quanh con vừa chạm (không có ⇒ giữ 1v1).
- Test +6 (`2v2_*`): 2 đấu sĩ chờ lệnh, `ActivesOf` đếm đúng, mục tiêu chọn trúng đúng con, AoE trúng cả
  2, `AdvanceRound` 2v2 không treo. **`Run M64 TurnBased Tests` 83/83.**

### ✅ PLAY-TEST 2026-09-03 (`CombatArena.unity`, 2v2 + băng ghế)
Roster: Kiếm sĩ Hỏa + Y tu (active) + Cung thủ Băng (dự bị) vs Linh lộc (Mộc) + Lang yêu. Xác nhận:
2 plate/phe + dòng "+ Tên HP%"; vòng chốt-lệnh **lặp qua cả 2 đấu sĩ phe ta**; chọn chiêu đơn-mục-tiêu
+ 2 địch → **panel chọn mục tiêu** (Linh lộc / Lang yêu / Quay lại); sát thương trúng ĐÚNG con đã chọn
(Đâm Mạnh → Linh lộc 117→49); log ghi **`⟪kém hiệu quả⟫`** (Mộc→Hoả); ép Kiếm sĩ Hỏa Hp→0.5 → ngã →
**`AwaitingForcedSwitch`, UI forced mode, msg "Kiếm sĩ Hỏa ngã xuống! Chọn đồng đội ra thay.", nút "Cung
thủ Băng" KHÔNG có "Quay lại"**; `SubmitForcedSwitch(0)` → Cung thủ Băng ra trận; chơi hết 5 vòng →
winner A → **XP: 2 survivor +26 mỗi con** (đúng công thức `8 + 6×3` chia 2, con ngã KHÔNG được XP),
lưu vào `CreatureLevelRegistry`; màn Result "THẮNG!". HP scale theo cấp đúng (Lv4 ×1.18, Lv16 ×1.9). 0 error.
⚠ Bẫy M36: Editor mất focus phiên MCP → bơm `drv.Tick()` + gọi `Update()` bằng reflection.

**CÒN NỢ (cập nhật):** ~~bảng khắc hệ~~ ✅ · ~~hệ level~~ ✅ · ~~prompt bắt đổi khi ngã~~ ✅ ·
~~phe địch 2v2/3v3~~ ✅ · (còn) M30 injury commit từ arena · seed arena hằng số · địch Wild 2v2 chưa
play-test (mới test scene demo).

### §14 — 3 cải tiến sau khi đọc Pokémon Showdown (2026-09-03, xem `M64_SHOWDOWN_STUDY.md`)

**(1) `SkillDefinition.priority` (int) → sort initiative** ("Quick Attack"). `CreatureDuelCombatDriver.
CommandPriority(actor)`: Switch/Flee = 6 (giải quyết trước mọi đòn), Skill = `skill.priority`, còn lại 0.
`ResolveRound` sort: `priority → speed → id`. Skill JSON cũ (priority 0) ⇒ 0 hồi quy. RTwP dùng
`castTime`, không đọc field này.

**(2) Tag `"ranged"` → bỏ luật hàng-trước** trong `TurnSkillView.RowRule` (ưu tiên hơn so chuỗi
`weaponType == "bow"`, giữ bow làm fallback). Idiom `flags` của Showdown — data-driven, không hardcode.

**(3) `SkillEffectRegistry` + `ISkillEffect` + `SkillDefinition.effects[]`** — tầng dispatch hiệu ứng
chiêu (bản `gBattleScriptsForMoveEffects`/`onHit`/`Battle::Move::0xx` làm theo kiểu Nizhangjie). Mirror
`BuildingFunctionRegistry` từng nét: static, đăng ký tường minh (`CoreSkillEffects.RegisterAll`), lazy
`EnsureDefaults`, `ResetForTests`. `Combat/Drivers/TurnBased/Effects/` — hiệu ứng RIÊNG của turn-based
driver (luật §4.4). Hook: `ModifyDamage(in ctx, current)` (chồng như `onBasePower`), `OnHit(in ctx)`
(drain/recoil/status), `OnAfterCast(in ctx)` (tự hồi). `SkillEffectBase` no-op để né default-interface-
method (IL2CPP). Wire: `ResolveHit` gọi `ApplySkillDamageModifiers` trước `DamageCalculator` +
`RunSkillEffects(OnHit)` sau khi trừ máu; `DoSkill` gọi `RunSkillEffects(OnAfterCast)` cuối.
**4 hiệu ứng lõi** (nội dung tự bật qua `effects[]`, chưa JSON nào dùng — luật D6):
`core:drain` (hồi value × dmg) · `core:recoil` (chịu value × dmg) · `core:heal_self` (hồi value × MaxHp,
OnAfterCast) · `core:status_onhit` (áp `TurnStatus` — bản data-driven của `TryBuildOnHit`; `param`=id,
`value`=lượt, `value2`=DoT/lượt).

Test +11 (`Prio_*` / `Tag_*` / `SEff_*` — priority move đi trước dù chậm hơn, RowRule theo tag, 4 hiệu
ứng đo trực tiếp + drain chạy qua driver). **`Run M64 TurnBased Tests` 94/94** · hồi quy `M64 CombatDriver`
25/25 · `WildCombat` 53/53.

**✅ PLAY-TEST 2026-09-03 (`CombatArena.unity` 2v2):** (2) `new TurnSkillView(rangedDef).RowRule == Any`,
`meleeDef == PrefersFront`. (3) gắn chiêu "Hút Máu" (`effects:[{core:drain,0.5}]`) cho Y tu, ép HP 700 →
đánh trúng 46 CHÍ MẠNG → log **"Y tu hút 23 máu"** → HP 700→723. (1) gắn chiêu "Chớp Nhoáng"
(`priority:2`, dmg 0.7) cho Y tu (SPD 86) → vòng sau Y tu **ra đòn TRƯỚC** Kiếm sĩ Hỏa (SPD 104) và cả
phe địch (Lang yêu SPD 121) — log: dòng đầu là "Y tu dùng Chớp Nhoáng!". 0 error. Bẫy M36 (no-focus):
plate HP-number không tự refresh, dòng "+ Tên %" thì có (đọc mỗi frame).

### §15 — đóng nợ "field ngữ nghĩa" (2026-09-03, schema khuôn Pokémon Showdown `MoveData`)

Sau khi đọc `sim/dex-moves.ts` của Showdown (MIT), đối chiếu `SkillDefinition` ↔ `MoveData`: khớp ~80%.
Thêm **6 field ngữ nghĩa** vào `SkillDefinition` (append-only, mặc định = "suy như cũ" ⇒ 42 skill JSON +
mọi test giữ NGUYÊN hành vi, **0 tiêu rng thêm** vì mọi nhánh mới có guard "chỉ chạy khi field ≠ mặc định"):

| Field | Mặc định | Ý nghĩa |
|---|---|---|
| `targetClass` | `""` | `one_enemy`/`all_enemies`/`random_enemy`/`one_ally`/`all_allies`/`self`. Rỗng ⇒ suy từ targetingMode/hitShape/affectsAllies |
| `rowRule` | `""` | `any`/`prefers_front`/`front_only`. Rỗng ⇒ suy (tag "ranged" / weaponType "bow" ⇒ any) |
| `charges` | `0` | PP thật. 0 ⇒ suy từ `skillTier` (weak 20 / medium 15 / strong·onhit 10) |
| `accuracy` | `1` | xác suất trúng 0..1. ≤0 ⇒ luôn trúng |
| `secondaryChance` | `1` | xác suất status on-hit kích. Chỉ gate `TryBuildOnHit` (hiệu ứng `effects[]` VẪN luôn chạy — chúng là cơ chế chính) |
| `hitCount` | `1` | multi-hit mỗi mục tiêu, trần 10 |

`TurnSkillView` mỗi getter đổi từ "suy" → **"đọc field trước, suy sau"**. Driver `DoSkill`: cuộn
`Accuracy` 1 lần/mục tiêu (chỉ khi <1) → "TRƯỢT!"; lặp `HitCount` lần `ResolveHit` → "(đánh N đòn)".
`ResolveHit`: `SecondaryChance` gate `TryBuildOnHit`. `actionCost` **cố ý bỏ** (turn-based luôn 1 lượt/đòn
— D6). Test +22 (`Sem_*` — field thắng suy, rỗng thì suy, back-compat chiêu thật, driver hitCount 3 +
accuracy 0.001 → TRƯỢT). **`Run M64 TurnBased Tests` 116/116** · hồi quy 25/25 + 53/53.

### §16 — STAB (Same-Type-Attack-Bonus) (2026-09-03)

Gap elemental rẻ nhất (`M64_SHOWDOWN_STUDY.md` §9): con dùng chiêu **trùng hệ của chính nó** → nhân
sát thương. Showdown gọi là STAB, cứng ×1.5; ở đây là **DATA**:

- `ElementChartConfig.stabMultiplier` (mặc định `1.5`; `Sanitize()` ép về 1.5 nếu mod đặt < 1 —
  "STAB < 1" là vô nghĩa). `element_chart.json` ×2 khai tường minh `"stabMultiplier": 1.5` cho modder
  thấy (field default cũng tự lo).
- `ElementChart.StabMultiplier` (property) + `ElementChart.Stab(int skillElement, int casterElement)` —
  trả `stabMultiplier` khi `skillElement > 0 && == casterElement`, ngược lại `1f`.
- `CreatureDuelCombatDriver.ResolveHit`: `float stab = (sk != null && actor.Def != null) ?
  ElementChart.Stab(element, actor.Def.elementAffinity) : 1f;` rồi gấp vào
  `Calculate(..., effMult * typeMult * lvlRatio * stab, ...)`. **CHỈ chiêu thật** — đánh thường
  (`sk == null`) không có hệ nên không STAB (mirror Struggle của Showdown). Log gắn `⟪cùng hệ⟫` khi
  `stab > 1f`.
- **M22/RTwP KHÔNG đụng** — STAB (như khắc hệ §13) là quyết định của driver, không phải Combat Model
  chung. Determinism giữ nguyên: `stab` là phép nhân thuần, không cuộn rng.
- Test +7 (`Stab_*` — config mặc định 1.5, `Stab(8,8)`/`Stab(3,8)`/`Stab(0,0)`/`Stab(8,0)`, Sanitize
  ép, driver `beast_deer` hệ 8 dùng `elem_wood_strike` ăn ~1.5× so với `elem_fire_strike`, bảng khắc
  hệ rỗng để cô lập STAB). **`Run M64 TurnBased Tests` 123/123** · hồi quy 25/25 + 53/53.

### §17 — BẬC CHỈ SỐ (stat stage −6..+6) (2026-09-04)

Buff/debuff kiểu Vũ Điệu Kiếm (+2 ATK) / Gầm Gừ (−1 ATK) / Uy Hiếp. **Chỉ ATK / DEF / SPD** (đủ cho ca
kinh điển; ACC/EVA để sau — chúng cần bảng riêng + nối vào cuộn accuracy). Trận turn-based nguyên tử
⇒ bậc thuần transient, **không vào save**.

- **`TurnCombatant`** += `_stages` (`Dictionary<StatType,int>`, chỉ tra theo khoá — không lặp, an toàn
  determinism) + `StageOf(s)` · `AddStage(s, delta)` (kẹp −6..+6, **trả về số bậc THỰC SỰ đổi** cho log
  "đã chạm trần/sàn") · `ResetStages()` · `StageMul(s)` (bảng kinh điển: `n≥0 → (2+n)/2`; `n<0 → 2/(2−n)`;
  +1→1.5 · +2→2 · +6→4 · −1→0.667 · −6→0.25).
- **`Speed`** nay `× StageMul(SPD)`. **`ResolveHit`** gấp `actor.StageMul(ATK) / tgt.StageMul(DEF)` vào
  hệ số `Calculate` (nhất quán với `lvlRatio` — xấp xỉ, KHÔNG rescale đường cong DEF ⇒ Combat Model chung
  nguyên vẹn).
- **`SwitchIn`** gọi `active.ResetStages()` cho con RỜI SÂN (mirror Pokémon: xuống băng ghế = mất buff).
- **Hiệu ứng `core:stat_stage`** (`CoreSkillEffects`, `OnHit`): `param` = "atk"/"def"/"spd"; `value` =
  số bậc (âm = giảm); `value2 ≥ 1` ⇒ áp cho CHÍNH MÌNH, khác ⇒ con bị đánh.
- **Chiêu status 0 sát thương** (Vũ Điệu Kiếm/Gầm Gừ): `DoSkill` có nhánh MỚI cho
  `!IsOffensive && !IsHeal && effects.Count > 0` — resolve mục tiêu rồi CHỈ chạy `effects[]` (không qua
  `DamageCalculator` ⇒ không cuộn crit, không chip 1 máu). Gate chặt `effects.Count > 0` ⇒ 0 hồi quy
  (chưa chiêu JSON nào khớp; summon không có `effects[]`).
- **Nợ:** AI (`PickAiCommand`) chỉ chọn chiêu `IsOffensive`/heal ⇒ **AI không tự buff** — để cho §AI
  (bước 6 lộ trình). Chưa có chiêu JSON status thật — chờ nội dung (D6).
- Test +20 (`Stage_*` — toán bậc + kẹp trần/sàn + reset + Speed theo bậc + `core:stat_stage` trực tiếp
  + driver ATK+2 ⇒ dmg ~×2 + status move áp bậc không sinh dòng damage + switch xoá bậc).
  **`Run M64 TurnBased Tests` 143/143** · hồi quy 25/25 + 53/53.

### §18 — HỆ TRẠNG THÁI ĐÚNG NGHĨA (bỏng/độc/tê/ngủ/băng) (2026-09-04)

Trước: `TurnStatus` chỉ có `DotPerTurn` cứng + `SkipsTurn` bool. Giờ đủ 6 loại kinh điển, **data-driven**.

- **`Data/combat/status_conditions.json`** (×2) + **`TurnStatusCatalog`** (`Combat/Drivers/TurnBased/`,
  mirror `ElementChart`: `EnsureLoaded`/`Rebuild`/`ResetForTests`/`TryGet`/`Has`/`Build(id, rng, turnsOverride)`).
  `TurnStatusConditionDef`: `dotFractionMax` · `dotFlat` · `escalating` (độc nặng nhân theo vòng) ·
  `speedMul` (tê 0.5) · `atkMul` (bỏng 0.5) · `skipChance` (tê 0.25, cuộn MỖI vòng, status GIỮ) ·
  `thawChance` (băng 0.2, cuộn cuối vòng ⇒ TAN) · `skipsTurn` (ngủ/băng) · `turns`/`minTurns`/`maxTurns`
  (ngủ 1–3 random).
- **`TurnStatus`** += `DotFractionMax` · `Escalating` · `ElapsedTurns` · `SpeedMul` · `AtkMul` ·
  `SkipChance` · `ThawChance`. `ApplyStatus` merge theo trục MẠNH hơn (mul lấy `Min`, chance lấy `Max`).
- **`TickStatusesEndOfRound(rng)`** (đổi chữ ký — chỉ driver gọi): DoT = cứng + %MaxHp × (luỹ tiến ?
  vòng : 1); thaw-roll gỡ sớm; **rng CHỈ tiêu khi `ThawChance > 0`** ⇒ 0 hồi quy determinism.
- **`TurnCombatant`** += `StatusSpeedMul`/`StatusAtkMul` (tích các mul) + `RollPartialSkip(rng)` (rng chỉ
  tiêu khi có `SkipChance > 0`). `Speed × StatusSpeedMul`. `ResolveHit` × `actor.StatusAtkMul` (bỏng —
  bước 3 phys/special sẽ giới hạn về đòn vật lý). `StartRound`: `Stunned` (đầy đủ) hoặc `RollPartialSkip`
  (một phần) ⇒ `Skip()`.
- **Hiệu ứng `core:ailment`** (MỚI, `CoreSkillEffects`): `param` = id loại, `value` = vòng ghi đè,
  `value2` = +DoT cứng. Param lạ ⇒ bỏ qua + log. **`core:status_onhit` giữ NGUYÊN** (flat, "tự chế").
  **`TryBuildOnHit` (đường derive `onHitEffectTag` cũ) giữ NGUYÊN** — burn/freeze/stun từ skill JSON cũ
  vẫn hành vi cũ (nợ: migrate sang `effects[]`/`core:ailment` để ăn atk-halve/luỹ tiến).
- Test +18 (`Ail_*` — catalog nạp, Build, DoT %MaxHp, độc nặng luỹ tiến 100/200/300, tê Speed ×0.5,
  ngủ Stunned, băng tự tan, RollPartialSkip ~25%, `core:ailment` áp + param lạ bỏ qua, driver bỏng
  hạ sát thương ra ~0.5×). **`Run M64 TurnBased Tests` 161/161** · hồi quy 25/25 + 53/53.

### §19 — PHYSICAL / SPECIAL SPLIT (MATK / MDEF) (2026-09-04) — user chọn "split thật"

Thêm **stat công/thủ PHÉP** vào core, split thật (không phải tag driver-local).

- **`StatType`** += `MATK`, `MDEF` (append-only, cuối enum — save-safe).
- **Derive:** `DiscipleFactory.DeriveCombatStats` (nguồn sự thật cho đệ tử/player) — `MATK = Ngộ Tính
  × 1.2`; `MDEF = Ngộ Tính × 0.4 + Foundation × 0.2`. Combat unit (`UnitFactory.BuildStats`): đọc
  `CombatUnitDefinition.matk`/`mdef` (field MỚI, default 0).
- **`DamageCalculator.Calculate(..., bool special = false)`** — `special` ⇒ đọc `MATK`/`MDEF`, và
  **fallback về ATK/DEF nếu stat phép ≤ 0** (mọi unit hiện có chưa khai matk ⇒ chiêu special vẫn chạy
  bằng ATK/DEF). **`special` mặc định false ⇒ MỌI caller cũ (RTwP/M22/mọi test) 0 thay đổi.**
- **`SkillDefinition.damageCategory`** ("physical"/"special"/"status", rỗng ⇒ suy: !tấn công &&
  !hồi ⇒ status, còn lại physical). `TurnSkillView.DamageCategory` (enum `TurnDamageCategory`).
- **`ResolveHit`** branch: special ⇒ `Calculate(special: true)` + bậc dùng track `MATK`/`MDEF` (không
  phải ATK/DEF) + **MIỄN đòn bỏng** (`StatusAtkMul` chỉ áp physical). `core:stat_stage` `param` nhận
  thêm "matk"/"mdef".
- **UI:** `PawnStatsPage.Order` chèn MATK/MDEF sau ATK/DEF; `DiscipleUIHelper` tên "Công Phép"/"Thủ
  Phép" + mô tả.
- **Cơ hội (không phải nợ):** các chiêu `elem_*` / `bang_phien_*` hiện là physical — content có thể
  đánh dấu `"damageCategory": "special"` để chúng ăn theo Ngộ Tính.
- Test +14 (`PS_*` — derive MATK/MDEF, `DamageCategory` field+suy, `Calculate` special dùng MATK +
  fallback khi MATK=0, driver special MATK cao ≫ physical ATK thấp, bậc MATK ăn / bậc ATK không ăn cho
  special, bỏng không hạ special, `core:stat_stage` param "matk"). **`Run M64 TurnBased Tests` 175/175**
  · hồi quy M64 CombatDriver 25/25, WildCombat 53/53, M Combat Skill 7/7, M31 Equipment 17/17, Creature
  Body 46/46, M2 6/6. (M Combat Core 3 FAIL = bug `DamageCalculator ×0.8` CÓ TRƯỚC — task_3e48645e.)

### §20 — ĐẶC TÍNH SINH VẬT (ICreatureAbility) (2026-09-04)

Passive bẩm sinh hook vào mốc cố định. Mirror `SkillEffectRegistry`/`CreatureAbilityRegistry` từng nét.

- **`Combat/Drivers/TurnBased/Abilities/`** — `ICreatureAbility` (6 hook) + `CreatureAbilityBase` (no-op,
  né DIM) + `AbilityContext` (Holder/Other/Opponents/Rng/DamageDealt/Log) + `CreatureAbilityRegistry`
  (static, EnsureDefaults→`CoreCreatureAbilities.RegisterAll`) + `CoreCreatureAbilities` (6 đặc tính).
- **Hook:** `OnEnterBattle` (vào sân) · `OnLeaveField` (rời sân) · `ModifyIncomingElementMult(atkElement)`
  · `ModifyOutgoingDamage` · `ModifyIncomingDamage` · `OnStruck` (Other = kẻ đánh).
- **6 đặc tính lõi:** `core:intimidate` (vào sân −1 ATK MỌI địch) · `core:levitate` (miễn hệ Thổ) ·
  `core:guts` (×1.5 sát thương ra khi dính trạng thái) · `core:thick_fat` (Hỏa/Băng nhận vào ×0.5) ·
  `core:sturdy` (đầy máu ⇒ sống 1 máu qua đòn hạ gục) · `core:static` (30% làm tê kẻ đánh trúng mình).
- **`CombatUnitDefinition.abilityId`** (field mới "") → `TurnCombatant.AbilityId` + `.Ability` (resolve
  lười, cache). `UnitFactory.BuildSide` + `ArenaCombatBridge.Make` set từ Def.
- **Driver wiring:** `FinishBegin` fire `OnEnterBattle` cho actives đầu trận; `SwitchIn` fire
  `OnLeaveField`(rời)→`OnEnterBattle`(vào); `ResolveHit` gọi 4 hook còn lại (element mult trước
  `LastEffect`, out/in-damage quanh `TakeDamage`, `OnStruck` sau — bỏ nếu tgt ngã).
- **0 hồi quy:** đặc tính opt-in qua `abilityId` (chưa JSON nào khai) ⇒ `Ability == null` ⇒ hook không
  chạy ⇒ rng y nguyên. Weather-dependent abilities (Drizzle…) HOÃN cùng hệ thời tiết.
- **Nợ:** pawn (PawnBody) chưa có đường lấy ability (từ race/trait — mở rộng sau); AI chưa "biết" đặc
  tính địch.
- Test +11 (`Ab_*` — registry, Uy Hiếp vào sân −1 ATK, Lơ Lửng miễn Thổ, Da Dày ½ Hỏa, Gan Lì ×1.5 khi
  dính độc, Trụ Vững sống 1 máu / không cứu khi không đầy máu, Tĩnh Điện làm tê qua `OnStruck`).
  **`Run M64 TurnBased Tests` 186/186** · hồi quy 25/25 + 53/53.

### §21 — VẬT PHẨM MANG THEO (ITurnHeldItem) (2026-09-04)

Vật passive mang vào trận (Đồ Ăn Thừa / Cầu Sinh Mệnh…). Mirror `ICreatureAbility` từng nét.

- **`Combat/Drivers/TurnBased/HeldItems/`** — `ITurnHeldItem` (4 hook) + `TurnHeldItemBase` (no-op) +
  `HeldItemContext` + `TurnHeldItemRegistry` + `CoreHeldItems`.
- **4 vật lõi:** `core:leftovers` (cuối vòng hồi MaxHp/16) · `core:life_orb` (sát thương ra ×1.3 +
  phản chấn 1/10 sát thương gây ra) · `core:focus_band` (đầy máu ⇒ sống 1 máu qua đòn hạ gục, VỠ —
  dùng 1 lần) · `core:power_charm` (×1.15, không hao mòn).
- **`TurnCombatant`** += `HeldItemId` · `HeldItemUsed` (vật một-lần) · `HeldItem` (resolve lười, trả
  null sau khi `HeldItemUsed`).
- **Driver:** `ResolveHit` out/in-damage + `OnDealtDamage` (sau `TakeDamage`); `ResolveRound` cuối
  vòng gọi `OnEndOfRound`.
- **⚠ KHÔNG qua `EquipmentFunctionRegistry`** (lệch gợi ý ban đầu, có lý do): vật mang theo KHÔNG có
  hành vi lúc-mặc ⇒ handler `IEquipmentFunction.OnEquip` sẽ là no-op = đúng anti-pattern doc
  `IBuildingFunction` cấm ("đừng vẽ handler trước khi có việc thật"). Thay vào đó:
  **`EquipmentDefinition.heldItemId`** (field data mới) + **`ArenaCombatBridge.ResolveHeldItemId`**
  (quét trang bị ĐANG MẶC của pawn qua `EquipmentManager.GetAllEquipped` → món đầu tiên khai
  `heldItemId`). `EquipmentFunctionRegistry` vẫn 0 handler — vẫn chờ ca thật "Pháp Bảo mở skill".
- **0 hồi quy:** opt-in qua `heldItemId` (chưa equipment JSON nào khai) ⇒ `HeldItem == null`.
- Test +11 (`HI_*` — registry, Đồ Ăn Thừa hồi/không hồi khi đầy, Cầu Sinh Mệnh ×1.3 + phản chấn,
  Bùa Lực ×1.15, Đai Cứu Mạng sống 1 máu → item null → lần 2 chết thật).
  **`Run M64 TurnBased Tests` 197/197** · hồi quy 25/25 + 53/53 + M31 Equipment 17/17.

### §22 — AI CHỌN CHIÊU / BUFF / MỤC TIÊU (M64 bước 6) (2026-09-04)

`PickAiCommand` trước: bốc chiêu tấn công ĐẦU TIÊN, không bao giờ buff, không bao giờ đổi con.

- **`ScoreAttack(actor, view, out target)`** — ước lượng sát thương KỲ VỌNG (không cuộn rng, không gọi
  `DamageCalculator`): `atkStat × Multiplier × HitCount × typeMult × STAB × (1 − DEF/(DEF+100))`, quét mọi
  con địch đang ra trận, chọn con điểm cao nhất; ×1.5 nếu ước tính đủ hạ gục ("kết liễu"). Chọn đúng
  ATK/MATK + bậc theo `DamageCategory`.
- **`PickAiCommand` mới:** ① heal (như cũ) → ② chấm điểm mọi chiêu tấn công + đánh thường, nhiễu nhỏ
  TẤT ĐỊNH (`0.9 + 0.2×rng`) → ③ mọi đòn đều "kém hiệu quả" (`< ATK×0.55`) + có dự bị + `switchRoll <
  AiSwitchChance` (0.25) ⇒ `Switch(0)` → ④ chưa lên bậc nào + `statusRoll < AiStatusMoveChance` (0.35) +
  có chiêu status có `effects[]` ⇒ dùng nó → ⑤ ra chiêu điểm cao nhất kèm `bestTarget` (2v2 nhắm đúng
  con bị khắc hệ), hoặc đánh thường.
- ⚠ **Determinism (bẫy A4):** số lần cuộn rng CHỈ phụ thuộc SỐ CHIÊU (cấu trúc), KHÔNG phụ thuộc trạng
  thái con — cuộn jitter/switchRoll/statusRoll TRƯỚC rồi mới quyết định. **Early-out:
  `Skills.Count == 0 ⇒ Attack() ngay, 0 cuộn rng`** (giữ nguyên chuỗi rng cho con "chỉ đánh thường" —
  nếu không, `Lvl_Driver_MultGapDoi` đỏ vì lệch vị trí cuộn crit giữa 2 lần chạy).
- `AiStatusMoveChance` / `AiSwitchChance` = field public trên driver (chỉnh / tắt = 0).
- Test +4 (`AI_*` — chọn chiêu khắc-hệ-cao khi có Thủy+Hỏa vs địch Mộc, dùng buff khi chưa lên bậc +
  không buff lại khi đã có bậc, 2v2 nhắm đúng con bị khắc hệ). **`Run M64 TurnBased Tests` 201/201** ·
  hồi quy M64 CombatDriver 25/25, WildCombat 53/53.

---

## ✅ Lộ trình "làm nốt vị Pokémon" (2026-09-04) — 6/6 XONG

1. Bậc chỉ số (§17) · 2. Hệ trạng thái đúng nghĩa (§18) · 3. Physical/Special split (§19) ·
4. Đặc tính sinh vật (§20) · 5. Vật phẩm mang theo (§21) · 6. AI chọn chiêu/buff/mục tiêu (§22).
**`Run M64 TurnBased Tests` 201/201** (từ 116).

### §23 — NHÓM A: 10 cơ chế "vị Pokémon căn bản" (2026-09-04) — `Run M64 TurnBased Tests` 242/242

Làm HẾT nhóm A ("game vị-Pokémon nào cũng gặp"). Mọi nhánh rng mới đều gate "chỉ cuộn khi field/state ≠
mặc định" ⇒ 42 skill JSON + toàn bộ test cũ chạy y nguyên (chỉ `Lvl_Driver_MultGapDoi` phải ép SPD để
cố định thứ tự lượt — nhiễu cũ, không phải hồi quy). Hồi quy: M64 CombatDriver 25/25, WildCombat 53/53,
M Combat Skill 7/7, M31 Equipment 17/17, Creature Body 46/46, M13 Hom 11/11. (M Combat Core 3 FAIL =
`DamageCalculator ×0.8` CÓ TRƯỚC — task_3e48645e, KHÔNG đụng.)

| # | Cơ chế | Chỗ chính |
|---|---|---|
| **A1** | Bậc CHÍNH XÁC / NÉ TRÁNH (bảng 3/(3±n)) | `TurnCombatant.AccEvaStageMul`; `EffectiveAccuracy(actor,tgt,v)` = acc × accStage ÷ evaStage; `core:stat_stage` param "acc"/"eva" |
| **A6** | Dao động sát thương 85–100% | `ResolveHit`: `dmg *= 0.85 + 0.15×rng` (1 cuộn/đòn) |
| **A2** | 1 trạng thái NẶNG + miễn nhiễm theo hệ | `TurnStatusConditionDef.major`/`immuneElements`; `ApplyStatus` trả `bool` (chặn nếu đã mang major khác); `TurnStatusCatalog.IsImmuneByElement`; bỏng↔Hỏa, tê↔Lôi, băng↔Băng |
| **A3** | RỐI LOẠN + CHÙN BƯỚC | catalog `confusion` (2–5 vòng, `selfHitChance` 0.33) + `TurnCombatant.Flinched`; driver `RollConfusionSelfHit` + check Flinched TRƯỚC `ExecuteCommand`; hiệu ứng `core:confuse` / `core:flinch` |
| **A4** | BẢO VỆ (Protect) — dùng liên tiếp ×1/3 | `TurnCombatant.Protected`/`ProtectStreak`; hiệu ứng `core:protect` (cuộn `1/3^streak`); `ResolveHit` chặn nếu `tgt.Protected` |
| **A5** | Chiêu 2-lượt (tích lực) + nạp năng lượng | `SkillDefinition.chargeTurns`/`mustRecharge`; `TurnCombatant.PendingChargeSkill`/`MustRecharge`; `StartRound` ép lệnh; `DoSkill` nhánh tích/tung |
| **A7** | Tự-đổi (U-turn) + ép-đổi (Roar) | hiệu ứng `core:self_switch` / `core:force_switch` → `TurnCombatant.WantsSelfSwitch`; driver quét sau execute-loop, `SwitchIn` bench[0] |
| **A8** | Đặc tính HÚT HỆ | hook MỚI `ICreatureAbility.OnNullifiedHit`; `core:water_absorb`/`volt_absorb` (hồi 25% MaxHp), `core:flash_fire`/`sap_sipper` (+1 bậc); `ResolveHit` early-return khi typeMult→0 do đặc tính |
| **A9** | THỜI TIẾT (Nắng/Mưa/Bão Cát/Mưa Đá) | `TurnWeather` enum + `driver.Weather`/`SetWeather`; `AbilityContext`/`SkillEffectContext` += `Action<string,int> Field` (ctor cũ giữ nguyên, `null`); `WeatherDamageMul` (Nắng ↑Hỏa ↓Thủy), chip cuối vòng, đếm lùi; hiệu ứng `core:set_weather` + đặc tính `core:drought`/`drizzle`/`sand_stream`/`snow_warning` |
| **A10** | ĐA HỆ (dual-type) | `CombatUnitDefinition.elementAffinity2` + `TurnCombatant.Element1`/`Element2`/`ElementOverride1/2`; `ElementChart.Multiplier(atk,def1,def2)` = nhân dồn; `Stab(el,c1,c2)` khớp bất kỳ; giữ overload cũ ⇒ M22/RTwP 0 đổi |
| **A11** | BẮT quái trong trận | `TurnCommandKind.Capture` + `TurnCommand.Capture(target)`; `driver.CaptureAllowed` (bridge bật cho trận hoang) + `CapturedCombatant`; `TryCapture` chance = 0.30 + (1−máu%)×0.50 + 0.15 nếu dính trạng thái. **Nợ: bridge chưa nối `CapturedCombatant` → `CompanionOps.Tame` (M29); `ArenaBattleUI` chưa có nút BẮT.** |

+41 test (`A1_*`…`A11_*`). Còn thiếu (nhóm B — chiều sâu PvP, single-player ít cần): bẫy sân, màn chắn,
Trick Room, Substitute, terrain, EV/IV/Nature, Choice-lock, họ Quả, contact-flag. Ngoài phạm vi: Mega/
Z-Move/Dynamax/Tera.

---

### §24 — CONTENT DEMO TRUNG TÍNH khai field mới (2026-09-04) — `Run M64 TurnBased Tests` 242/242

Sau §17-§23 mọi cơ chế đã có, nhưng **CHƯA có file JSON nào khai** `damageCategory` / `effects[]` mới /
`priority` / `chargeTurns` / `mustRecharge` / `hitCount` / `abilityId` / `elementAffinity2` / `heldItemId`
— cơ chế chạy trong test synthetic mà không có content thật chứng minh JsonUtility deserialize đúng, và
`CombatArena.unity` không có gì để play-test các vị đó.

**Chốt với user: content demo = TRUNG TÍNH, không xianxia** (đúng §5.8 CLAUDE.md "Core Content Pack trung
tính"). Engine là general; ngũ hành chỉ là bộ element hiện nạp trong `element_chart.json`. **KHÔNG tải data
Pokémon** (chép biểu đạt có bản quyền = vi phạm kể cả để test/không phát hành). Dải số (base power 40–120,
PP 5–35, accuracy 0.6–1.0) lấy từ schema `moves.ts` của Pokémon Showdown (MIT) — không chép mon cụ thể.

- **16 skill `demo_*`** (`Data/skills/`, ×2 folder) — mỗi cái khai ĐÚNG 1-2 field mới:
  `demo_quick_jab` (priority 2) · `demo_ember` (special + `secondaryChance` 0.1 + `core:ailment` burn) ·
  `demo_focus_beam` (`chargeTurns` 1) · `demo_overload_blast` (`mustRecharge`) · `demo_guard`
  (`core:protect`, `targetClass` self, priority 3) · `demo_honing` (`core:stat_stage` tự buff +2) ·
  `demo_snarl` (`core:stat_stage` debuff địch −1) · `demo_venom_fang` (`core:ailment` poison) ·
  `demo_hypno_pulse` (`core:ailment` sleep + `accuracy` 0.6) · `demo_bewilder` (`core:confuse`) ·
  `demo_gale_dash` (`core:self_switch`) · `demo_bellow` (`core:force_switch`, `priority` −6) ·
  `demo_sun_call` / `demo_rain_call` (`core:set_weather`) · `demo_twin_strike` (`hitCount` 2) ·
  `demo_drain_kiss` (`core:drain` 0.5).
- **6 combat_unit `demo_*`** (`Data/combat_units/`, ×2) khai `abilityId` + `matk`/`mdef` + (một số)
  `elementAffinity2`: `demo_striker` (player, guts) · `demo_pyre_hound` (Hỏa, flash_fire) ·
  `demo_storm_raptor` (Phong/Lôi, volt_absorb) · `demo_bog_toad` (Thủy/Mộc, water_absorb) ·
  `demo_bulwark_ox` (Thổ, sturdy) · `demo_frost_stag` (Băng/Phong, snow_warning).
- **12 skill CŨ đánh dấu `damageCategory: "special"`** — 8 `elem_*_strike` + `skl_hoa_cau`/`bang_phong`/
  `loi_kich`/`thuy_tien` (đòn dẫn linh khí = MATK vs MDEF). **Chỉ `CreatureDuelCombatDriver` đọc field
  này — RTwP/M22 y nguyên.** 2 test helper (`DmgAtkStage`/`DmgWithBurn`) từng hardcode `elem_fire_strike`
  để kiểm nhánh PHYSICAL — đổi sang `SkillDefinition` synthetic `damageCategory:"physical"` (test bậc
  ATK / bỏng KHÔNG được phụ thuộc category của content JSON).
- **4 equipment** (`art_spirit_gourd`→leftovers, `jew_jade_pendant`→power_charm, `jew_blood_bead`→
  life_orb, `jew_frost_bell`→focus_band) khai `heldItemId` — `ArenaCombatBridge.ResolveHeldItemId` quét
  đồ đang mặc.
- Hồi quy: M64 CombatDriver 25/25, WildCombat 53/53, M Combat Skill 7/7, Creature Body 46/46, M31
  Equipment 17/17, M34 EquipmentSlot 58/58, M53 Memory Save 34/34. (M Combat Core 3 FAIL CÓ TRƯỚC.)
- **NỢ ĐÃ BIẾT (không thuộc phạm vi đợt này):** `Assets/StreamingAssets/Data/skills/` thiếu ~12 file
  skill legacy (`bang_phien_*`/`cung_phap_*`/`hoa_kiem_*`/`kbp_*`/`loi_thuong_*`/`phong_am_*`/
  `tho_daikien_*`/`thuy_quyen_*`) — drift 2-folder CÓ TRƯỚC session này, chỉ ảnh hưởng BUILD thật (Editor
  đọc `Assets/Data`). Demo content mới + 12 file đã sửa ĐỀU đã mirror. **CHƯA verify Play Mode
  `CombatArena.unity`** với các demo unit.

---

## 11. FEATURE_CATALOG — thêm khi làm xong

Thêm mục vào CUỐI `Assets/Scripts/FEATURE_CATALOG_2026-08-27.md`:
`Combat Mode Framework (M64)` — công dụng: chọn lối đánh lúc tạo ván (bộ chọn trên MainPanel), khung
`ICombatDriver` cắm nhiều paradigm (RTwP / turn-based hàng-trước-sau / …) trên cùng `Data/skills` +
Combat Model; code: `Combat/Drivers/`, `Combat/Model/`, `Combat/Modes/CombatDriverRegistry`,
`Data/combat_modes/`, `SaveData.combatModeId`, scene `CombatSandbox.unity` → `CombatArena.unity`.
