# M7 — Crafting & World Events

**Milestone:** Crafting systems (Alchemy + Forge), World Event simulation, Disciple Relationship system, BuildingController efficiency fix.

---

## Cây thư mục mới

```
Assets/Scripts/
├── Sim/
│   ├── Crafting/                        ← NEW (M7)
│   │   ├── RecipeRegistry.cs
│   │   ├── CraftResult.cs
│   │   ├── CraftingEngine.cs            (static utility)
│   │   ├── AlchemySystem.cs
│   │   └── ForgeSystem.cs
│   ├── World/                           ← NEW (M7)
│   │   ├── WorldEvent.cs
│   │   └── WorldEventSystem.cs
│   └── Disciples/
│       └── RelationshipSystem.cs        ← NEW (M7)
├── LocalMap/
│   ├── LocalMapManager.cs               ← Updated (M7: 16-step tick)
│   └── Buildings/
│       └── BuildingController.cs        ← Fixed (M7: per-buildingId efficiency)
├── Core/Events/
│   └── SampleEvents.cs                  ← Extended (8 new M7 events)
└── Tests/Editor/
    └── M7SimTest.cs                     ← NEW (10 bài test)
```

---

## 1. RecipeRegistry

**File:** `Sim/Crafting/RecipeRegistry.cs`

Lưu trữ và tra cứu `RecipeDefinition` (POCO đã có từ trước).

```csharp
var registry = new RecipeRegistry();
registry.Register(new RecipeDefinition {
    recipeId                 = "pill_qi_gather",
    recipeType               = "pill",
    comprehensionRequired    = 40,
    comprehensionForHighQuality = 70,
    inputs = new List<ResourceCost> {
        new ResourceCost { resourceId = ResourceType.LingCao, amount = 3 }
    },
    outputId         = "pill_qi_gather",
    baseOutputAmount = 1,
});

// Query
var pillRecipes = registry.GetByType("pill");
var available   = registry.GetAvailableFor(disciple, "pill");  // lọc theo Comprehension đủ điều kiện
```

---

## 2. CraftResult & CraftQuality

**File:** `Sim/Crafting/CraftResult.cs`

```csharp
public enum CraftQuality { Failed = -1, Common = 0, Uncommon = 1, Rare = 2, Legendary = 3 }

public readonly struct CraftResult
{
    public bool         Success;
    public CraftQuality Quality;
    public string       OutputId;
    public int          OutputAmount;
    public string       ByproductId;      // null nếu không có
    public float        ByproductAmount;
}
```

**Byproduct** (chỉ khi Quality ≥ Rare):

| RecipeType  | Byproduct Resource     |
|-------------|------------------------|
| pill        | ThaoDuocHiem (+1.0)    |
| equipment   | GoHiem (+0.5)          |
| talisman    | LinhKhi (+2.0)         |

---

## 3. CraftingEngine (static utility)

**File:** `Sim/Crafting/CraftingEngine.cs`

### Công thức thành công

```
baseChance     = 0.60
statBonus      = (stat - 50) × 0.005           // +0.5% mỗi điểm trên 50
tierBonus      = (buildingTier - 1) × 0.05     // +5% mỗi tier trên 1
successChance  = Clamp(baseChance + statBonus + tierBonus, 0.05, 0.95)
```

### Bảng Quality

| Điều kiện                                       | Quality   |
|-------------------------------------------------|-----------|
| Random fail                                     | Failed    |
| stat < comprehensionForHighQuality              | Common    |
| stat ≥ comprehensionForHighQuality, roll < 0.15 | Legendary |
| stat ≥ comprehensionForHighQuality, roll < 0.40 | Rare      |
| stat ≥ comprehensionForHighQuality, roll < 0.70 | Uncommon  |
| còn lại                                         | Common    |

### Ghi chú quan trọng

- **Nguyên liệu bị tiêu thụ ngay cả khi thất bại** — tạo sức ép quản lý resource.
- `outputAmount = baseOutputAmount` (Common/Uncommon) hoặc `+1` (Rare/Legendary).

### Usage

```csharp
bool ok = CraftingEngine.HasIngredients(recipe, sect.Resources);
CraftResult result = CraftingEngine.AttemptCraft(
    recipe, crafter, sect.Resources,
    primaryStat: StatType.Comprehension,
    buildingTier: 2
);
```

---

## 4. AlchemySystem

**File:** `Sim/Crafting/AlchemySystem.cs`

Xử lý recipe loại `"pill"`. Stat chính: **Comprehension**.

```csharp
var alchemy = new AlchemySystem(recipeRegistry, discipleRegistry, globalBus);

bool ok = alchemy.AttemptCraft(
    discipleId: "disciple_zhao",
    recipeId:   "pill_qi_gather",
    resources:  sect.Resources
);
// Publish AlchemyCraftedEvent khi xong
```

**Điều kiện từ chối:**
- Recipe không tồn tại hoặc không phải `"pill"`
- Disciple không tồn tại / đã chết
- `Comprehension < recipe.comprehensionRequired`
- Nguyên liệu không đủ

---

## 5. ForgeSystem

**File:** `Sim/Crafting/ForgeSystem.cs`

Xử lý recipe loại `"equipment"` (stat: **Force**) và `"talisman"` (stat: **Comprehension**).

```csharp
var forge = new ForgeSystem(recipeRegistry, discipleRegistry, globalBus);

bool ok = forge.AttemptCraft(
    discipleId: "disciple_li",
    recipeId:   "sword_iron",
    resources:  sect.Resources
);
// Publish ForgeCraftedEvent khi xong
```

---

## 6. WorldEventSystem

**File:** `Sim/World/WorldEventSystem.cs`

### Cơ chế spawn

- **RollIntervalDays** = 30 ngày
- **BaseEventChance** = 40%
- **MaxConcurrentEvents** = 2

Mỗi 30 ngày, roll 1 lần. Nếu thành công → chọn ngẫu nhiên 1 event từ pool (không trùng event đang active).

### 6 sự kiện tích hợp sẵn

| EventId              | Loại              | Thời gian | Hiệu ứng chính                                            |
|----------------------|-------------------|-----------|-----------------------------------------------------------|
| `drought`            | natural_disaster  | 20 ngày   | linh_dien production ×0.5, linh_cao price ×1.5           |
| `spirit_tide`        | spiritual         | 15 ngày   | luyen_vo production ×1.3, ling_khi daily gain +2         |
| `beast_tide`         | combat            | 10 ngày   | ling_khi drain -5/ngày, tăng quest combat spawn          |
| `sect_war`           | political         | 30 ngày   | training ×1.2, khai_khoang ×0.8, sect_war quest bonus    |
| `plague_beast`       | natural_disaster  | 25 ngày   | khai_khoang ×0.7, ling_thach price ×1.4, drain ling_cao  |
| `merchant_festival`  | economic          | 7 ngày    | market price ×0.85 (giảm), tăng trade quest spawn        |

### Production multiplier

```csharp
float mult = WorldEventSystem.GetProductionMult("linh_dien_field");
// drought đang active → trả về 0.5
// spirit_tide đang active → 1.0 (không ảnh hưởng linh_dien)
// cả hai đang active → 0.5
```

### Market pressure

Mỗi ngày event active, system gọi `MarketDynamics.RecordSale()` / `RecordPurchase()` để dịch chuyển giá thị trường tự nhiên.

---

## 7. RelationshipSystem

**File:** `Sim/Disciples/RelationshipSystem.cs`

### 3 loại quan hệ

| Loại         | Tạo                               | Hiệu ứng                                                      |
|--------------|-----------------------------------|---------------------------------------------------------------|
| MasterStudent | `AddMasterStudent(masterId, studentId)` | Học trò đột phá → thầy +3 loyalty; Thầy chết → trò -20 loyalty |
| Brotherhood   | `AddBrotherhood(idA, idB)`         | Cùng building → +3% efficiency; Bị thương → partner -5 loyalty; Chết → partner -10 loyalty |
| Rivalry       | `AddRivalry(idA, idB)`             | Đối thủ đột phá → -5 loyalty; Mỗi 30 ngày → `RivalryTrainingTriggeredEvent` |

### Event bridges (auto-wired qua GlobalEventBus)

| Event nhận                  | Hành động                                           |
|-----------------------------|-----------------------------------------------------|
| `RealmBreakthroughEvent`    | Master +3 loyalty; Rival -5 loyalty (ganh tị)      |
| `DiscipleDiedEvent`         | Student -20 loyalty; Brotherhood survivor -10       |
| `DiscipleInjuredEvent`      | Brotherhood partner -5 loyalty (thương cảm)        |

### Brotherhood efficiency bonus

```csharp
float bonus = RelationshipSystem.GetBrotherhoodEfficiencyBonus("disciple_zhao");
// +0.03 nếu partner còn sống và cùng CurrentBuildingId != null
// 0.0 nếu không có brotherhood hoặc khác building
```

---

## 8. BuildingController Fix (M7)

**File:** `LocalMap/Buildings/BuildingController.cs`

### Vấn đề (trước M7)

`ComputeAverageWorkerEfficiency()` không dùng dict `efficiencies` được truyền vào — luôn fallback về `count / MaxWorkers × 1.0`.

### Fix

```csharp
private float ComputeAverageWorkerEfficiency(
    Building building,
    Dictionary<string, float> efficiencies)
{
    // M7: lookup pre-aggregated entry theo buildingId
    if (efficiencies.TryGetValue(building.BuildingId, out float preAgg))
        return preAgg;

    // Legacy fallback (khi chưa có WorkAssignment wired)
    int count = building.CurrentWorkers;
    if (count == 0) return 0f;
    return (float)count / Mathf.Max(1, building.MaxWorkers);
}
```

---

## 9. LocalMapManager — 16-step Tick

**File:** `LocalMap/LocalMapManager.cs`

```
Tick()
├── 1.  DiscipleRegistry.OnDayPassed()            — aging + loyalty drift
├── 2.  SalarySystem.OnDayPassed()                — lương tháng
├── 3.  BuildingController.Tick()                 — production (dùng BuildWorkerEfficiencies())
├── 4.  QuestRegistry.OnDayPassed()               — deadline + auto-complete
├── 5.  DispatchSystem.OnDayPassed()              — countdown + injury apply
├── 6.  MarketDynamics.OnDayPassed()              — price drift
├── 7.  Shop.OnDayPassed()                        — shop restock / income
├── 8.  Sect.OnDayPassed()                        — linh khí regen passive
├── 9.  StorytellerSystem.OnDayPassed()           — refresh quests mỗi 7 ngày
├── 10. TraitUnlockSystem.OnDayPassed()           — check AND condition unlock
├── 11. RankPromotion.CheckPromotion()            — thăng cấp tông môn
├── 12. AgingSystem.OnDayPassed()          (M6)   — aging milestones + near-death
├── 13. InjurySystem.OnDayPassed()         (M6)   — recovery countdown
├── 14. CompetitorMarketPool.OnDayPassed() (M6)   — NPC competitor activity
├── 15. WorldEventSystem.OnDayPassed()    (M7)    — world events tick + spawn
├── 16. RelationshipSystem.OnDayPassed()  (M7)    — rivalry tick mỗi 30 ngày
└──     _globalBus.Publish(DayPassedEvent)
```

### BuildWorkerEfficiencies() — per-buildingId

```csharp
// Trả về Dictionary<buildingId, aggregatedEfficiency>
// Mỗi building:
//   eff = BuildingEfficiencyTable.ComputeEfficiency(disciple, building)  // stat + loyalty + tier
//       × AgingSystem.GetAgeEfficiencyMultiplier(disciple)               // M6
//       + RelationshipSystem.GetBrotherhoodEfficiencyBonus(discipleId)   // M7
//   avgEff = totalEff / MaxWorkers
//   result[buildingId] = avgEff × WorldEventSystem.GetProductionMult(building.Definition.id)  // M7
```

---

## 10. M7 Events mới (8 sự kiện)

**File:** `Core/Events/SampleEvents.cs`

| Event                          | Fields chính                                                                 |
|--------------------------------|------------------------------------------------------------------------------|
| `AlchemyCraftedEvent`          | DiscipleId, RecipeId, OutputId, Amount, Quality(string), Success             |
| `ForgeCraftedEvent`            | DiscipleId, RecipeId, OutputId, Amount, Quality(string), Success, RecipeType |
| `WorldEventStartedEvent`       | EventId, EventName, EventType, DurationDays                                  |
| `WorldEventEndedEvent`         | EventId, EventName                                                           |
| `RelationshipBonusEvent`       | SourceId, BeneficiaryId, RelType, LoyaltyDelta, Reason                       |
| `RivalryTrainingTriggeredEvent`| DiscipleIdA, DiscipleIdB, DaysRival                                          |

_(AlchemyCraftedEvent và ForgeCraftedEvent kết hợp với 2 crafting systems; 4 còn lại từ World + Relationship.)_

---

## 11. Test Suite (M7SimTest)

**File:** `Tests/Editor/M7SimTest.cs`  
**Menu:** `Nizhangjie/Run M7 Tests`

| # | Tên test                                  | Kiểm tra                                               |
|---|-------------------------------------------|--------------------------------------------------------|
| 1 | `HasIngredients_ReturnsFalse_WhenEmpty`   | RecipeRegistry + CraftingEngine.HasIngredients         |
| 2 | `IngredientsConsumedOnFailure`            | Nguyên liệu bị trừ ngay cả khi craft thất bại         |
| 3 | `AlchemySystem_Success_AddsOutput`        | AlchemySystem craft thành công → SectResources tăng   |
| 4 | `AlchemySystem_LowComprehension_Rejected` | Comprehension không đủ → trả về false, resources nguyên |
| 5 | `ForgeSystem_PublishesForgeCraftedEvent`  | ForgeSystem publish ForgeCraftedEvent đúng fields      |
| 6 | `RecipeRegistry_GetByType_Filters`        | GetByType("pill") chỉ trả về pill recipes              |
| 7 | `WorldEventSystem_SpawnsEvent`            | Sau ≤100 ngày → ít nhất 1 WorldEventStartedEvent       |
| 8 | `WorldEventSystem_ProductionMult_Active`  | GetProductionMult trả về <1.0 khi drought active       |
| 9 | `RelationshipSystem_MasterStudent_Bonus`  | RealmBreakthroughEvent → thầy +3 loyalty               |
|10 | `RelationshipSystem_Brotherhood_Efficiency`| Cùng building → GetBrotherhoodEfficiencyBonus = 0.03  |

---

## 12. M7 Checklist

- [x] `RecipeRegistry` — Register, Get, GetByType, GetAvailableFor
- [x] `CraftResult` / `CraftQuality` enum
- [x] `CraftingEngine` — HasIngredients, AttemptCraft (success formula, quality roll, byproduct)
- [x] `AlchemySystem` — pill crafting, Comprehension gate, AlchemyCraftedEvent
- [x] `ForgeSystem` — equipment/talisman crafting, ForgeCraftedEvent
- [x] `WorldEvent` / `WorldEventInstance` POCOs
- [x] `WorldEventSystem` — 6 built-in events, spawn mechanics, production mult, market pressure
- [x] `RelationshipSystem` — 3 types, event bridges (breakthrough/death/injury)
- [x] `BuildingController` fix — per-buildingId aggregated efficiency lookup
- [x] `LocalMapManager` — 16-step tick, BuildWorkerEfficiencies với aging+brotherhood+world mult
- [x] `SampleEvents.cs` — 8 new M7 events
- [x] `M7SimTest.cs` — 10 bài test
- [x] `M7_README.md` — tài liệu này

---

## M8+ Pending

- **Romantic relationships** — partner buff, jealousy mechanic, co-departure when loyalty = 0
- **Crafting queue** — multi-day craft jobs, building occupancy
- **Recipe discovery** — unlock recipes qua quest hoặc research
- **Quality-scaled item stats** — Rare pill gives stronger realm breakthrough bonus
- **WorldEvent → StorytellerSystem** — world events spawn themed quests automatically
- **Rival sect AI** — CompetitorMarket giờ chỉ trade; M8+ thêm rival sect send disciples on quests
- **Relationship UI** — hiển thị quan hệ trong Disciple panel
