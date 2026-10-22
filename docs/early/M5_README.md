# M5 — Sim Systems

**Namespace:** `Nizhangjie.LocalMap.*` + `Nizhangjie.Sim.*`

M5 hoàn thiện vòng lặp quản lý tông môn: LocalMapManager kết nối tất cả hệ thống, thêm per-building efficiency, rank promotion, market dynamics, procedural storyteller, thiên kiếp, và trait unlock tự động.

---

## Cấu trúc thư mục (thêm mới M5)

```
LocalMap/
├── LocalMapManager.cs            — MonoBehaviour orchestrator: tick toàn bộ sim mỗi ngày
└── Buildings/
    └── BuildingEfficiencyTable.cs — Per-building stat mapping cho production efficiency

Sim/
├── Sect/
│   └── SectRankPromotion.cs      — Kiểm tra điều kiện lên rank + publish event xác nhận
├── Disciples/
│   ├── HeavenlyTribulation.cs    — Rủi ro thiên kiếp khi đột phá từ Realm 2 trở lên
│   └── TraitUnlockSystem.cs      — Unlock hidden trait theo điều kiện (realm/loyalty/age...)
├── Economy/
│   └── MarketDynamics.cs         — Supply/demand pressure + mean reversion
└── Quests/
    └── StorytellerSystem.cs      — Sinh quest tự động dựa trên world state (7 ngày/lần)

Core/Events/
└── SampleEvents.cs               — +10 event mới M5 (Rank, Tribulation, Trait, Quest events)

Tests/Editor/
└── M5SimTest.cs                  — 10 tests qua menu Nizhangjie/Run M5 Tests
```

---

## Core concepts

### LocalMapManager — bộ điều phối trung tâm

```csharp
// LocalMapManager là MonoBehaviour duy nhất M5 thêm.
// Tất cả hệ thống khác là plain C# — owned bởi Manager này.

public class LocalMapManager : MonoBehaviour
{
    public string SettlementId = "settlement_main";
    public float  DaysPerRealSecond = 1f;   // 0 = pause, 5 = fast forward

    // Owned systems (set up trong Awake)
    public Sect Sect;
    public DiscipleRegistry DiscipleRegistry;
    public SalarySystem     SalarySystem;
    public TraitUnlockSystem TraitUnlockSystem;
    public HeavenlyTribulation Tribulation;
    public BuildingController  BuildingController;
    public WorkAssignment      WorkAssignment;
    public GridSystem          Grid;
    public Market              Market;
    public Shop                Shop;
    public MarketDynamics      MarketDynamics;
    public QuestRegistry       QuestRegistry;
    public QuestBoard          QuestBoard;
    public DispatchSystem      DispatchSystem;
    public StorytellerSystem   StorytellerSystem;
    public SectRankPromotion   RankPromotion;

    public long TotalDays { get; private set; }
}

// Thứ tự tick mỗi ngày (LocalMapManager.Tick):
// 1.  DiscipleRegistry.OnDayPassed       — aging, loyalty drift
// 2.  SalarySystem.OnDayPassed           — trả lương mỗi 30 ngày
// 3.  BuildingEfficiency rebuild cache   — tính lại per-worker efficiency
// 4.  BuildingController.Tick            — production → SectResources
// 5.  MarketDynamics.OnDayPassed         — price drift + mean reversion
// 6.  QuestRegistry.OnDayPassed          — deadline countdown, auto-complete
// 7.  QuestBoard.OnDayPassed             — expire cũ
// 8.  DispatchSystem.OnDayPassed         — countdown dispatch + resolve
// 9.  StorytellerSystem.OnDayPassed      — sinh quest mới mỗi 7 ngày
// 10. TraitUnlockSystem.OnDayPassed      — kiểm tra unlock hidden traits
// 11. RankPromotion.CheckPromotion       — kiểm tra điều kiện lên rank
//    + publish DayPassedEvent
```

### BuildingEfficiencyTable — per-building stat mapping

```csharp
// Mỗi loại building dùng stat khác nhau của đệ tử làm primary stat.
// Công thức: efficiency = primaryStat / StatCap × loyaltyFactor
// loyaltyFactor = loyalty / 100
// Clamp: [0, 1.5]

// Mapping BuildingDefinition.id → primary stat:
Building ID prefix    Stat dùng
─────────────────────────────────────────────────────
linh_dien            Foundation          (cánh đồng linh thảo)
khai_khoang          Force               (khai thác khoáng thạch)
luyen_dan            (Comprehension + Agility) / 2  (luyện đan)
ve_bua               Comprehension       (vẽ bùa chú)
re_khi_cu            (Force + Comprehension) / 2    (rèn khí cụ)
luyen_vo             (Force + Agility) / 2          (luyện võ)
thuong_diem          Charisma            (tiệm buôn)
[default]            (Agility + Foundation) / 2

// Usage:
float eff = BuildingEfficiencyTable.ComputeEfficiency(disciple, building);
// building phải đang SetActive() và có ít nhất 1 worker slot
```

### MarketDynamics — giá thị trường

```csharp
// MarketDynamics theo dõi supply/demand pressure per resource type.
// Áp lực độc lập decay mỗi ngày về 0; price multiplier mean-revert về 1.0

var dynamics = new MarketDynamics(market);
market.SetDynamics(dynamics);   // wiring: Buy/Sell auto-gọi RecordPurchase/RecordSale

// Khi player bán nhiều:
dynamics.RecordSale(ResourceType.LingCao, 200f);   // supply pressure tăng → giá giảm
// Khi player mua nhiều:
dynamics.RecordPurchase(ResourceType.Dan, 50f);    // demand pressure tăng → giá tăng

// Mỗi ngày:
dynamics.OnDayPassed();
//   1. decay supplyPressure × (1 - DecayRate=0.10)
//   2. decay demandPressure × (1 - DecayRate)
//   3. priceMult -= supplyPressure × SupplyImpact=0.02
//   4. priceMult += demandPressure × DemandImpact=0.02
//   5. mean reversion: priceMult += (1 - priceMult) × MeanReversionRate=0.05
//   6. clamp priceMult [0.4, 2.5]
//   7. sync → Market (set base price × priceMult)

// Spread: sell price = buy price × 0.65 (market margin 35%)
float price = dynamics.GetCurrentPrice(ResourceType.LingCao);
```

### StorytellerSystem — sinh quest tự động

```csharp
// Refresh mỗi 7 ngày. Số quest mỗi lần: Random(1, 3).
// Ưu tiên theo world state:

var storyteller = new StorytellerSystem(questRegistry, board, globalBus);

// Priority logic mỗi lần refresh:
// 1. LingThach < 500  → Trading quest khẩn cấp (reward LingThach cao)
// 2. LingCao < 100    → Delivery quest (giao linh thảo)
// 3. Prestige ≥ 60    → HighTier: MonsterHunt khó hoặc Dungeon
// 4. TotalDays % 30 == 0 → Story Beat (4 beat tuần hoàn — xem bảng)
// 5. Padding          → Random: Bandit (prestige<40) / MonsterHunt (prestige≥40)

// Tick mỗi ngày:
storyteller.OnDayPassed(sect, discipleRegistry, totalDays);
// → khi đủ 7 ngày: sinh quest mới + AddToBoard + publish QuestGeneratedEvent

// QuestBoard.AvailableCount để check số quest hiện có
int available = board.AvailableCount;
```

#### Story Beats (tuần hoàn mỗi 30 ngày)

| Beat | Tên | QuestType | Difficulty | Reward đặc biệt |
|---|---|---|---|---|
| 0 | Bí Thư Cổ Đại | Exploration | 30 | 1 CongPhap |
| 1 | Thương Đoàn Cầu Viện | Bandit | 40 | 2 KhiCu |
| 2 | Yêu Thú Đột Biến | MonsterHunt | 65 | 3 YeuDan |
| 3 | Phái Môn Thách Đấu | Punitive | 70 | 100 LinhKhi |

### HeavenlyTribulation — thiên kiếp

```csharp
// Thiên kiếp bắt đầu từ Realm 2 (Kim Đan). Realm 0-1 luôn thành công.

var trib = new HeavenlyTribulation(globalBus);

// Attempt breakthrough:
bool success = trib.Attempt(disciple, targetRealm: 2, sect.Resources,
                            danBonusPower: 50f,      // đan dược boost combat power
                            buaResistReduction: 0.2f); // bùa giảm tribulation strength 20%

// Công thức:
// tribStrength  = targetRealm × 150 × Random(0.8, 1.2) × (1 - buaResistReduction)
// combatPower   = (ATK + DEF×0.5 + MaxHP×0.1) × loyalty/100 + danBonusPower
// successChance = power / (power + tribStrength×0.5)   clamp [0.1, 0.9]

// Kết quả:
// roll < chance          → Thành công: realm++, MaxLifespan += 50×realm
//                           publish RealmBreakthroughEvent
// roll ≥ chance, không deviation (75%) → Thất bại nhẹ: loyalty -15
// roll ≥ chance, deviation (25%)       → Tẩu hỏa nhập ma: loyalty -40
//   35%: chết → DiscipleDiedEvent CauseTag="deviation"
//   65%: sống nhưng realm--

// Dù kết quả nào: publish TribulationStartedEvent lúc bắt đầu
// Thất bại: publish TribulationFailedEvent
```

### TraitUnlockSystem — hidden traits

```csharp
// Unlock hidden trait (isHidden=true) khi condition thoả.
// Check mỗi ngày, cache những trait đã unlock (không check lại).

var unlockSys = new TraitUnlockSystem(registry, globalBus, traitPool);

// Inject prestige nếu dùng condition "prestige >= X":
unlockSys.SetSectPrestige(sect.SectPrestige);

// Tick:
unlockSys.OnDayPassed();
// → foreach disciple Alive() → check từng TraitDefinition isHidden=true
//   → EvaluateCondition → AddTrait + publish TraitUnlockedEvent

// Cú pháp unlockCondition trong TraitDefinition:
"realm >= 2"       // CultivationRealm >= 2
"loyalty >= 90"    // Loyalty >= 90
"age >= 50"        // Age >= 50
"age <= 20"        // tuổi trẻ
"prestige >= 60"   // SectPrestige >= 60
// (kills, dispatch: M6+)

// Operators hỗ trợ: >= <= > < ==
// Không có AND/OR — M6+
```

### SectRankPromotion — lên hạng tông môn

```csharp
// Kiểm tra prerequisites → publish RankPromotionAvailableEvent
// Player confirm → PromoteRank() thực sự lên rank

var promo = new SectRankPromotion(globalBus);

// Daily check (có debounce 7 ngày):
promo.CheckPromotion(sect, discipleRegistry, buildingController);
// → đủ điều kiện → publish RankPromotionAvailableEvent { CurrentRank, NextRank }

// Player confirm (gọi từ UI):
bool promoted = promo.PromoteRank(sect, discipleRegistry, buildingController);
// → trừ 10% LingThach phí thủ tục
// → sect.SetRank(newRank), sect.AddPrestige(10f)
// → publish RankPromotedEvent
// Cooldown 30 ngày giữa các lần promote

// Prerequisites per rank:
```

#### Bảng điều kiện lên rank

| Rank hiện tại | Đệ tử | LingThach | Prestige | Công trình T3 | Công trình T4 | Đệ tử Realm≥3 |
|:---:|---:|---:|---:|:---:|:---:|:---:|
| 0 → 1 | 5  | 1,000   | 20 | 0 | 0 | 0 |
| 1 → 2 | 15 | 5,000   | 40 | 1 | 0 | 0 |
| 2 → 3 | 30 | 20,000  | 60 | 3 | 0 | 1 |
| 3 → 4 | 60 | 100,000 | 80 | 5 | 5 | 3 |

---

## Sự kiện mới M5

```csharp
// ── Rank ───────────────────────────────────────────────────────────────
RankPromotionAvailableEvent { CurrentRank, NextRank }   // đủ điều kiện, đợi confirm
RankPromotedEvent           { SectId, OldRank, NewRank } // đã promote

// ── Thiên Kiếp ─────────────────────────────────────────────────────────
TribulationStartedEvent  { DiscipleId, TargetRealm, TribStrength, CombatPower, SuccessChance }
TribulationFailedEvent   { DiscipleId, TargetRealm, IsDeviation }
// (success → RealmBreakthroughEvent — đã có từ M1)

// ── Traits / Quests ─────────────────────────────────────────────────────
TraitUnlockedEvent  { DiscipleId, TraitId }
QuestGeneratedEvent { QuestId, QuestType }
```

---

## Breakthrough flow

```
Player: AttemptBreakthrough("disciple_01", targetRealm: 2)
   ↓
LocalMapManager.AttemptBreakthrough()
   ↓
HeavenlyTribulation.Attempt(disciple, realm=2, resources)
   ↓
   publish TribulationStartedEvent (UI panel hiện lên)
   ↓
   [roll]
   ├── Thành công → realm=2, lifespan+100 → publish RealmBreakthroughEvent
   ├── Thất bại nhẹ  → loyalty-15
   └── Tẩu hỏa nhập ma → loyalty-40
       ├── Chết (35%)  → DiscipleDiedEvent ("deviation")
       └── Sống (65%)  → realm-- → publish TribulationFailedEvent (isDeviation=true)
```

---

## Test suite M5

Chạy qua **Nizhangjie → Run M5 Tests** (không cần Play Mode).

| # | Test | Kiểm tra |
|---|---|---|
| 1 | BuildingEfficiencyTable | linh_dien → Foundation stat, kết quả trong [0.3, 0.9] |
| 2 | SalarySystem | Tick 30 ngày → LingThach giảm từ 200 |
| 3 | MarketDynamics | Bán 200 LingCao → giá sau < giá trước |
| 4 | DispatchSystem | StartDispatch + tick 1 ngày → ActiveCount = 0 (resolved) |
| 5 | StorytellerSystem | Tick 7 ngày → board.AvailableCount ≥ 1 |
| 6 | SectRankPromotion | 5 đệ tử + 2000 LingThach + prestige 80 → event fired |
| 7 | HeavenlyTribulation | Realm 0→1 không có kiếp → luôn thành công |
| 8 | TraitUnlockSystem | Condition "realm >= 2" → unlock "hidden_veteran" |
| 9 | DiscipleRegistry | DiscipleLeftSectEvent → Unregister |
| 10 | SectResources.SpendMultiple | Dan thiếu → rollback, LingThach không đổi |

---

## Checklist M5

**LocalMap**
- [x] `LocalMap/LocalMapManager.cs` — MonoBehaviour orchestrator, 11-bước tick mỗi ngày
- [x] `LocalMap/Buildings/BuildingEfficiencyTable.cs` — per-building stat mapping, 7 loại + default

**Sim / Sect**
- [x] `Sim/Sect/SectRankPromotion.cs` — 4 cấp rank, debounce check, cooldown promote, phí 10%

**Sim / Disciples**
- [x] `Sim/Disciples/HeavenlyTribulation.cs` — formula đầy đủ, 3 outcome, bridge DiscipleDiedEvent
- [x] `Sim/Disciples/TraitUnlockSystem.cs` — 5 field, 5 operator, per-disciple unlock cache

**Sim / Economy**
- [x] `Sim/Economy/MarketDynamics.cs` — supply/demand pressure, decay, mean reversion, price sync
- [x] `Sim/Economy/Market.cs` — wired SetDynamics, Buy/Sell auto-gọi RecordPurchase/RecordSale

**Sim / Quests**
- [x] `Sim/Quests/StorytellerSystem.cs` — 7-ngày refresh, 5 trigger condition, 4 story beats, padding
- [x] `Sim/Quests/QuestBoard.cs` — thêm `AvailableCount` property

**Events**
- [x] `Core/Events/SampleEvents.cs` — +10 event: RankPromotionAvailable, RankPromoted, TribulationStarted, TribulationFailed, TraitUnlocked, QuestGenerated

**Tests**
- [x] `Tests/Editor/M5SimTest.cs` — 10 tests qua menu Nizhangjie/Run M5 Tests

---

## Pending (M6+)

- [ ] Breakthrough UI (TribulationStartedEvent → panel xác nhận + VFX)
- [ ] Rank promotion UI (RankPromotionAvailableEvent → dialog confirm)
- [ ] Story beat quest chain: quest A unlock quest B
- [ ] TraitUnlockSystem: AND/OR condition, scripted condition từ JSON
- [ ] Kill counter + Dispatch counter cho trait condition
- [ ] Market: competitor NPC settlement sinh supply/demand riêng
- [ ] BuildingEfficiencyTable: Tier modifier (Tier 2+ → +10% per tier)
- [ ] Dispatch injury system: disciple HP damage sau dispatch thất bại
- [ ] Disciple aging → tử vong tự nhiên (MaxLifespan reached)
