# M8 — Crafting Depth

**Milestone:** Crafting Queue (multi-day jobs), Recipe Discovery, Quality-Scaled Item Stats.

---

## Cây thư mục mới / thay đổi

```
Assets/Scripts/
├── Sim/Crafting/
│   ├── CraftJob.cs              ← NEW (M8) — job POCO
│   ├── CraftingQueue.cs         ← NEW (M8) — per-building queue + day tick
│   ├── RecipeDiscovery.cs       ← NEW (M8) — khám phá công thức
│   ├── ItemInstance.cs          ← NEW (M8) — item với quality-scaled stats
│   ├── CraftResult.cs           ← Updated (M8) — thêm QualityScore float
│   ├── CraftingEngine.cs        ← Updated (M8) — ResolveCraft() + QualityToScore()
│   ├── AlchemySystem.cs         ← Updated (M8) — discovery gate + EnqueueCraft()
│   └── ForgeSystem.cs           ← Updated (M8) — discovery gate + EnqueueCraft()
├── LocalMap/LocalMapManager.cs  ← Updated (M8) — 18-step tick
├── Core/Events/SampleEvents.cs  ← Updated (M8) — 3 new events
└── Tests/Editor/M8SimTest.cs    ← NEW (10 bài test)
```

---

## 1. CraftJob

**File:** `Sim/Crafting/CraftJob.cs`

POCO lưu trạng thái một công việc chế tạo:

| Field            | Ý nghĩa                                               |
|------------------|-------------------------------------------------------|
| `JobId`          | GUID 8 ký tự (duy nhất)                               |
| `RecipeId`       | Công thức cần làm                                     |
| `CrafterId`      | Disciple đang thực hiện                               |
| `BuildingId`     | Building đang chiếm                                   |
| `PrimaryStatType`| Stat roll khi hoàn thành (Comprehension hoặc Force)   |
| `BuildingTier`   | Tier lúc enqueue (ảnh hưởng success chance)           |
| `DaysRequired`   | Số ngày cần (`craftingHours / 24`, tối thiểu 1)       |
| `DaysProgress`   | Số ngày đã tiến hành                                  |
| `IsCancelled`    | Job bị huỷ (ingredient không hoàn trả)                |
| `IsActive`       | Đang ở slot active (không phải hàng chờ)              |
| `IsComplete`     | `DaysProgress >= DaysRequired && !IsCancelled`        |

---

## 2. CraftingQueue

**File:** `Sim/Crafting/CraftingQueue.cs`

### Model

```
Building A: [ActiveJob: pill_01 d=2/3] → [WaitQueue: sword_01, talisman_02]
Building B: [rảnh]
```

- Mỗi building có **1 active slot** — chỉ 1 job đang tiến hành.
- Các job kế tiếp xếp hàng (`Queue<CraftJob>`), được promoted tự động khi active slot giải phóng.

### Ingredient lifecycle

```
Enqueue()    → ConsumeIngredients() ngay  (khóa resource)
OnDayPassed() → khi IsComplete → ResolveCraft() → add output to resources
Cancel()     → ingredients KHÔNG hoàn trả
```

### API

```csharp
// Wire vào AlchemySystem / ForgeSystem
alchemy.SetCraftingQueue(craftingQueue);
forge.SetCraftingQueue(craftingQueue);

// Enqueue — trả về CraftJob hoặc null nếu thất bại
CraftJob job = alchemy.EnqueueCraft("pill_qi_gather", "disciple_zhao", building, resources);

// Query
bool   busy  = craftingQueue.IsBuildingBusy("building_alchemy_01");
CraftJob act = craftingQueue.GetActiveJob("building_alchemy_01");
int    wait  = craftingQueue.GetQueueCount("building_alchemy_01");

// Cancel (ingredient không hoàn trả)
craftingQueue.CancelJob(job.JobId);

// Tick — gọi từ LocalMapManager.Tick() step 16
craftingQueue.OnDayPassed(sect.Resources);
```

### Sự kiện

| Event                    | Khi nào                                      |
|--------------------------|----------------------------------------------|
| `CraftJobStartedEvent`   | Job trở thành active (lần đầu hoặc promoted) |
| `CraftJobCompletedEvent` | Job hoàn thành (kể cả thất bại roll)         |

---

## 3. RecipeDiscovery

**File:** `Sim/Crafting/RecipeDiscovery.cs`

### Cơ chế mở khoá

| Con đường   | Mô tả                                                                  |
|-------------|------------------------------------------------------------------------|
| Quest reward | `GrantDiscovery(discipleId, recipeId, "quest")` — instant unlock      |
| Research     | Roll tự nhiên mỗi ngày, dựa trên Comprehension (OnDayPassed)          |
| Event        | Sự kiện thế giới / NPC dạy công thức                                   |

### Research formula

```
baseChance  = 0.01  (1% / ngày / công thức chưa biết)
statBonus   = (Comprehension − 50) × 0.001
totalChance = Clamp(baseChance + statBonus, 0.001, 0.10)
```

- Comprehension = 50  → 1%/ngày  → trung bình ~100 ngày/công thức
- Comprehension = 100 → 6%/ngày  → trung bình ~17 ngày/công thức
- Comprehension = 150 → 10%/ngày → trung bình ~10 ngày/công thức (cap)

### API

```csharp
// Cấp phát (từ quest, event)
discovery.GrantDiscovery("disciple_zhao", "recipe_sword_iron", "quest");

// Kiểm tra trước khi cho craft
bool canCraft = discovery.IsDiscovered("disciple_zhao", "recipe_sword_iron");

// Lấy tất cả recipe đã biết
IReadOnlyCollection<string> known = discovery.GetDiscovered("disciple_zhao");

// Tick — gọi từ LocalMapManager.Tick() step 17
discovery.OnDayPassed(recipeRegistry, discipleRegistry);
```

### Gate trong AlchemySystem / ForgeSystem

```csharp
// Nếu discovery được wire → recipe chưa khám phá sẽ bị từ chối
alchemy.SetDiscovery(discovery);
forge.SetDiscovery(discovery);

// AttemptCraft() và EnqueueCraft() đều kiểm tra discovery
var result = alchemy.AttemptCraft("recipe_unknown", discipleId, building, resources);
// → CraftResult.Failure nếu chưa discovered, resources nguyên vẹn
```

---

## 4. ItemInstance + ItemQualityTier

**File:** `Sim/Crafting/ItemInstance.cs`

### Quality tier từ QualityScore

| QualityScore | Tier        | StatMultiplier |
|-------------|-------------|----------------|
| `< 0.50`    | Poor        | × 0.70 (−30%) |
| `0.50–0.75` | Normal      | × 1.00         |
| `0.75–0.95` | Good        | × 1.30 (+30%) |
| `≥ 0.95`    | Masterpiece | × 1.75 (+75%) |

### QualityScore sinh ra như thế nào

CraftingEngine sinh `QualityScore` từ `CraftQuality` (band roll):

| CraftQuality | QualityScore range    | → Tier phổ biến |
|--------------|-----------------------|-----------------|
| Failed       | 0.00                  | (no item)       |
| Common       | Random [0.10, 0.50)   | Poor / Normal   |
| Uncommon     | Random [0.50, 0.75)   | Normal          |
| Rare         | Random [0.75, 0.95)   | Good            |
| Legendary    | Random [0.95, 1.00]   | Masterpiece     |

### Tạo ItemInstance từ CraftResult

```csharp
// Trong ForgeSystem sau khi craft thành công:
var item = ItemInstance.FromCraftResult(
    result:   craftResult,
    recipeId: "recipe_sword_iron",
    crafterId: "disciple_li",
    baseATK: 120f,
    baseDEF: 30f
);

Debug.Log(item);
// → [Item 3f7a9c2b] output_sword_iron (Good, score=0.82, ATK=156.0, DEF=39.0)

// Stat query
float effectiveATK = item.EffectiveATK;  // BaseATK × StatMultiplier
float effectiveDEF = item.EffectiveDEF;
```

### Static helpers

```csharp
ItemQualityTier tier = ItemInstance.GetTier(0.78f);          // → Good
float mult           = ItemInstance.GetStatMultiplier(tier);  // → 1.30f
```

---

## 5. CraftingEngine — cập nhật (M8)

### ResolveCraft() — tách rời consume + resolve

```csharp
// Dùng bởi CraftingQueue khi job hoàn thành.
// Ingredient đã bị consume lúc Enqueue() → chỉ roll thành công/quality/output
CraftResult result = CraftingEngine.ResolveCraft(recipe, crafter, resources, StatType.Comprehension, tier);
```

### QualityScore trong CraftResult

`CraftResult` nay có thêm field `float QualityScore` — phạm vi [0, 1], sinh ra bởi `QualityToScore(CraftQuality)`.

```csharp
var result = alchemy.AttemptCraft(...);
float score = result.QualityScore;                     // 0.0 – 1.0
var tier    = ItemInstance.GetTier(result.QualityScore); // Poor/Normal/Good/Masterpiece
```

---

## 6. LocalMapManager — 18-step Tick

```
Tick()
├── 1–15. (giống M7, xem M7_README)
├── 16. M8: CraftingQueue.OnDayPassed()   — advance + complete jobs
├── 17. M8: RecipeDiscovery.OnDayPassed() — research roll tự nhiên
└── 18. Broadcast DayPassedEvent
```

### Wiring tự động trong InitSystems()

```csharp
CraftingQueue   = new CraftingQueue(RecipeRegistry, DiscipleRegistry, _globalBus);
RecipeDiscovery = new RecipeDiscovery(_globalBus);
AlchemySystem.SetCraftingQueue(CraftingQueue);
AlchemySystem.SetDiscovery(RecipeDiscovery);
ForgeSystem.SetCraftingQueue(CraftingQueue);
ForgeSystem.SetDiscovery(RecipeDiscovery);
```

---

## 7. M8 Events mới (3 sự kiện)

| Event                    | Fields chính                                                         |
|--------------------------|----------------------------------------------------------------------|
| `CraftJobStartedEvent`   | JobId, RecipeId, CrafterId, BuildingId, DaysRequired                 |
| `CraftJobCompletedEvent` | JobId, RecipeId, CrafterId, BuildingId, OutputId, QualityScore, QualityTier, Success |
| `RecipeDiscoveredEvent`  | DiscipleId, RecipeId, Source ("research"\|"quest"\|"event")          |

---

## 8. Test Suite (M8SimTest)

**File:** `Tests/Editor/M8SimTest.cs`  
**Menu:** `Nizhangjie/Run M8 Tests`

| # | Tên test                                              | Kiểm tra                                            |
|---|-------------------------------------------------------|-----------------------------------------------------|
| 1 | `CraftingQueue_Enqueue_BuildingBecomesBusy`           | IsBuildingBusy() = true sau enqueue                 |
| 2 | `CraftingQueue_OneDayAdvance_ProgressIs1`             | DaysProgress = 1 sau 1 lần OnDayPassed              |
| 3 | `CraftingQueue_JobComplete_PublishesCraftJobCompletedEvent` | CraftJobCompletedEvent fire khi xong          |
| 4 | `CraftingQueue_SecondJob_GoesToWaitQueue`             | GetQueueCount = 1 khi building đang bận             |
| 5 | `CraftingQueue_SecondJobPromoted_AfterFirstCompletes` | Job 2 trở thành active sau khi job 1 xong           |
| 6 | `RecipeDiscovery_NotDiscovered_ByDefault`             | IsDiscovered = false trước khi grant                |
| 7 | `RecipeDiscovery_Grant_IsDiscovered_And_FiresEvent`   | IsDiscovered = true + RecipeDiscoveredEvent fire    |
| 8 | `RecipeDiscovery_UndiscoveredRecipe_BlocksCraft`      | AttemptCraft() trả về Failure, resources nguyên     |
| 9 | `ItemInstance_QualityScore_MapsToCorrect4Tiers`       | Mapping + boundary đúng 4 tier                      |
|10 | `ItemInstance_StatMultiplier_Poor70pct_Masterpiece175pct` | EffectiveATK: Poor=70, Masterpiece=175 với base=100 |

---

## 9. M8 Checklist

- [x] `CraftJob` POCO — state + DaysProgress tracking
- [x] `CraftingQueue` — enqueue, promote, cancel, OnDayPassed
- [x] `RecipeDiscovery` — GrantDiscovery, research roll, IsDiscovered gate
- [x] `ItemInstance` + `ItemQualityTier` — 4 tier, stat multiplier
- [x] `CraftResult` — thêm `QualityScore` float
- [x] `CraftingEngine` — `ResolveCraft()` tách rời, `QualityToScore()`
- [x] `AlchemySystem` — `SetDiscovery()`, `SetCraftingQueue()`, `EnqueueCraft()`
- [x] `ForgeSystem` — `SetDiscovery()`, `SetCraftingQueue()`, `EnqueueCraft()`
- [x] `Building` — convenience constructor `(id, def, tier)` cho tests
- [x] `LocalMapManager` — 18-step tick, wiring M8 systems
- [x] `SampleEvents.cs` — 3 new M8 events
- [x] `M8SimTest.cs` — 10 bài test
- [x] `M8_README.md` — tài liệu này

---

## M9+ Pending

- **Item equip system** — gắn ItemInstance vào Disciple, áp dụng stat bonus vào CharacterStats
- **Enchantment slot** — thêm modifier đặc biệt vào item (drop từ monster, mua ở market)
- **Shared recipe (thầy dạy trò)** — master-student relationship → transfer 1 recipe/10 ngày
- **Hidden recipes** — cần vật phẩm đặc biệt + ritual để mở khóa
- **Crafting XP** — mỗi lần craft thành công → tăng craft skill level → giảm DaysRequired
- **Batch crafting** — 1 job tạo ra nhiều đơn vị output (hiện tại: luôn là 1 batch)
- **Building queue UI** — hiển thị CraftingQueue trong LocalMap panel
