# M6 — Disciple Depth

**Namespace:** `Nizhangjie.Sim.*` + `Nizhangjie.LocalMap.*`

M6 làm đệ tử "sống" hơn: aging milestones, thương tích thực với stat debuff, counters track kill/dispatch, trait condition AND, quest chain, và NPC competitor market tạo biến động độc lập.

---

## Cấu trúc thư mục (thêm mới M6)

```
Sim/
├── Disciples/
│   ├── AgingSystem.cs       — Milestones tuổi tác, LifeStage, near-death warning
│   └── InjurySystem.cs      — Thương tích từ dispatch: debuff stat, recovery countdown
├── Quests/
│   └── QuestChain.cs        — QuestChain + QuestChainRegistry: A → B unlock on complete
└── Economy/
    └── CompetitorMarket.cs  — NPC competitor tạo supply/demand pressure độc lập

Core/Events/SampleEvents.cs  — +8 event mới M6
Tests/Editor/M6SimTest.cs    — 10 tests qua menu Nizhangjie/Run M6 Tests
```

### Files cập nhật M6

```
Sim/Disciples/Disciple.cs        — +KillCount, +DispatchCount, RecordKill(), RecordDispatch()
Sim/Disciples/TraitUnlockSystem.cs — AND operator, wired kills/dispatch fields
Sim/Quests/DispatchSystem.cs     — wire InjurySystem, RecordDispatch(), QuestCompletedEvent
LocalMap/Buildings/BuildingEfficiencyTable.cs — +TierBonusPerTier (×1.10/tier)
LocalMap/LocalMapManager.cs      — +AgingSystem, +InjurySystem, +QuestChainRegistry,
                                   +CompetitorMarketPool, tick sequence 14 bước
```

---

## Core concepts

### AgingSystem — vòng đời đệ tử

```csharp
// AgingSystem bổ sung milestone events và cảnh báo — logic chết do tuổi
// vẫn nằm trong DiscipleRegistry.OnDayPassed().

var aging = new AgingSystem(discipleRegistry, globalBus);
aging.OnDayPassed();   // check tất cả alive disciples

// → Mỗi 10 năm tuổi → publish DiscipleAgedEvent { DiscipleId, Age, Milestone, LifeStage }
// → Khi còn < 10% MaxLifespan → publish DiscipleNearDeathByAgeEvent (1 lần duy nhất)

// LifeStage phân loại theo Age / MaxLifespan:
AgingSystem.GetLifeStage(disciple)
//  < 25% → YoungAdult (+5% năng suất — AgingSystem.GetAgeEfficiencyMultiplier)
//  25-50% → MiddleAge  (×1.00)
//  50-80% → Elder      (×0.90)
//  > 80%  → Ancient    (×0.80)

// Multiplier có thể nhân vào BuildingEfficiency (M7+: wired vào BuildingController)
float mult = AgingSystem.GetAgeEfficiencyMultiplier(disciple);
```

### InjurySystem — thương tích

```csharp
// Áp thương tích lên đệ tử — debuff stat, recovery theo ngày.

var injury = new InjurySystem(discipleRegistry, globalBus);

// Áp thương tích (từ DispatchSystem.ApplyDispatchInjury):
injury.ApplyInjury(discipleId, InjurySeverity.Moderate);
// → Giảm ATK/DEF/Foundation × EfficiencyPenalty (flat)
// → Publish DiscipleInjuredEvent { DiscipleId, Severity, RecoveryDays }

// Tick recovery:
injury.OnDayPassed();
// → Khi DaysLeft ≤ 0 → xóa modifier → Publish DiscipleRecoveredEvent

// Severity config:
//   Minor    — -10% stat, 5  ngày hồi
//   Moderate — -25% stat, 14 ngày hồi
//   Severe   — -50% stat, 30 ngày hồi

// Priority: Severe không bị ghi đè bởi Minor/Moderate (chỉ ghi đè nếu mới nặng hơn)
bool injured = injury.IsInjured(discipleId);
InjurySeverity? sev = injury.GetInjurySeverity(discipleId);

// DispatchSystem wiring (tự động nếu SetInjurySystem đã gọi):
dispatchSystem.SetInjurySystem(injury);
// → PartialSuccess: 1 đệ tử random → Minor injury
// → Failure:        1 đệ tử random → Severe injury (hoặc permadeath 20%)
```

### Disciple counters

```csharp
// KillCount + DispatchCount được track trực tiếp trên Disciple.

disciple.RecordKill(count: 5);    // thêm 5 kill
disciple.RecordDispatch();        // hoàn thành 1 dispatch (PartialSuccess trở lên)

int kills    = disciple.KillCount;
int dispatch = disciple.DispatchCount;

// DispatchSystem gọi RecordDispatch() tự động sau mỗi Success/PartialSuccess.
```

### TraitUnlockSystem AND (M6)

```csharp
// Hỗ trợ AND operator trong unlockCondition.
// Cú pháp: "condition1 AND condition2 AND condition3"

var traitDef = new TraitDefinition
{
    id              = "veteran_dispatcher",
    displayName     = "Lão Luyện Phái Đội",
    isHidden        = true,
    unlockCondition = "realm >= 2 AND dispatch >= 5 AND loyalty >= 80",
};

// Tất cả điều kiện phải đúng cùng lúc.
// Fields hỗ trợ M6: realm, loyalty, age, prestige, kills, dispatch
// Operators: >= <= > < ==

// Ví dụ phức hợp:
"realm >= 3 AND kills >= 100"       // Sát Thần Trait: cảnh giới cao + đã giết nhiều
"age <= 20 AND loyalty >= 90"       // Thiên Tài Trẻ: tuổi nhỏ nhưng trung thành
"dispatch >= 10 AND kills >= 50"    // Lão Binh Chinh Chiến
```

### QuestChain — chuỗi quest

```csharp
// Tạo chain từ danh sách questId theo thứ tự.
// Khi quest[i] Complete → quest[i+1] tự động unlock lên board.

// 1. Tạo tất cả quests và register vào QuestRegistry
questRegistry.Register(questA);
questRegistry.Register(questB);
questRegistry.Register(questC);

// 2. Post quest đầu tiên lên board
board.AddQuest(questA);

// 3. Tạo chain
var chain = new QuestChain(
    chainId:    "chain_ancient_relic",
    chainName:  "Bí Tích Cổ Đại",
    orderedQuestIds: new List<string> { "q_relic_01", "q_relic_02", "q_relic_03" });

var chainReg = new QuestChainRegistry(questRegistry, board, globalBus);
chainReg.RegisterChain(chain);

// 4. Flow: Player hoàn thành "q_relic_01"
//    → DispatchSystem publish QuestCompletedEvent("q_relic_01")
//    → QuestChainRegistry nghe → chain.AdvanceStep()
//    → board.AddQuest("q_relic_02") + publish QuestChainStepUnlockedEvent
//    → ...
//    → Sau khi "q_relic_03" xong → QuestChainCompletedEvent

// Query:
chain.CurrentStep      // step hiện tại (0-based)
chain.CurrentQuestId   // quest ID đang active
chain.IsComplete       // true nếu hết chain
```

### CompetitorMarket — NPC thị trường

```csharp
// Competitor tạo supply/demand pressure lên MarketDynamics chung.
// Không cạnh tranh trực tiếp với player — chỉ làm giá biến động.

var competitor = new CompetitorMarket(
    competitorId:       "thương_đoàn_bạch_lộc",
    competitorName:     "Thương Đoàn Bạch Lộc",
    preferredResources: new List<string> { ResourceType.LingCao, ResourceType.Dan },
    activityLevel:      0.6f);  // 0-1: càng cao giao dịch càng nhiều/lớn

var pool = new CompetitorMarketPool();
pool.Add(competitor);
pool.OnDayPassed(marketDynamics, playerPrestige: sect.SectPrestige);
// → mỗi ngày có 30% × activityLevel × prestigeMod khả năng giao dịch
// → nếu giao dịch: random Buy hoặc Sell, random resource trong prefs
// → RecordPurchase / RecordSale → giá thay đổi

// Competitor prestige scaling:
// playerPrestige=80 → competitor 1.15× hoạt động (market sôi hơn)
// playerPrestige=20 → competitor 0.85× (ít chú ý tông môn nhỏ)
```

### BuildingEfficiencyTable — Tier bonus (M6)

```csharp
// Tier 2+ building cho efficiency cao hơn (+10% per tier above 1).
// Phản ánh cơ sở vật chất tốt hơn (đồ gỗ tốt, linh văn tăng cường...).

// Tier 1 → ×1.00 (baseline)
// Tier 2 → ×1.10
// Tier 3 → ×1.20
// Tier 4 → ×1.30
// Tier 5 → ×1.40

float eff = BuildingEfficiencyTable.ComputeEfficiency(disciple, building);
//  = statEff × loyaltyMod × tierMultiplier

// Chỉ tính stat efficiency (không nhân tier):
float effBase = BuildingEfficiencyTable.ComputeEfficiencyBase(disciple, building);
```

---

## LocalMapManager tick sequence (M6 — 14 bước)

```
Bước  1. DiscipleRegistry.OnDayPassed       — aging, loyalty drift, natural death
Bước  2. SalarySystem.OnDayPassed           — lương tháng (mỗi 30 ngày)
Bước  3. BuildingController.Tick            — production → SectResources
Bước  4. QuestRegistry.OnDayPassed         — deadline countdown, auto-complete
Bước  5. DispatchSystem.OnDayPassed        — countdown dispatch, resolve + injuries
Bước  6. MarketDynamics.OnDayPassed        — price drift + mean reversion
Bước  7. Shop.OnDayPassed                  — NPC customer spawn
Bước  8. Sect.OnDayPassed                  — linh khí regen
Bước  9. StorytellerSystem.OnDayPassed     — sinh quest mới mỗi 7 ngày
Bước 10. TraitUnlockSystem.OnDayPassed     — check hidden trait unlock (AND support)
Bước 11. RankPromotion.CheckPromotion      — kiểm tra lên rank
Bước 12. AgingSystem.OnDayPassed           — milestone events, near-death warning
Bước 13. InjurySystem.OnDayPassed         — recovery countdown, xóa debuff
Bước 14. CompetitorMarketPool.OnDayPassed  — NPC competitor giao dịch
      + DayPassedEvent broadcast
```

---

## Sự kiện mới M6

```csharp
// ── Aging ────────────────────────────────────────────────────────────────
DiscipleAgedEvent            { DiscipleId, Age, Milestone, LifeStage(string) }
DiscipleNearDeathByAgeEvent  { DiscipleId, RemainingYears, MaxLifespan }

// ── Injury ───────────────────────────────────────────────────────────────
DiscipleInjuredEvent         { DiscipleId, Severity, RecoveryDays }
DiscipleRecoveredEvent       { DiscipleId, Severity }

// ── Quest Chain ───────────────────────────────────────────────────────────
QuestCompletedEvent          { QuestId, QuestType }      // bổ sung cho chain trigger
QuestChainStepUnlockedEvent  { ChainId, StepIndex, QuestId }
QuestChainCompletedEvent     { ChainId, ChainName }

// ── Competitor Market ─────────────────────────────────────────────────────
CompetitorTradeEvent         { CompetitorId, Resource, Amount, IsBuy }
```

---

## Test suite M6

Chạy qua **Nizhangjie → Run M6 Tests** (không cần Play Mode).

| # | Test | Kiểm tra |
|---|---|---|
| 1  | AgingSystem milestone | Tick qua tuổi 10 → DiscipleAgedEvent Milestone=10 |
| 2  | AgingSystem near-death | Age=95 MaxLifespan=100 → DiscipleNearDeathByAgeEvent |
| 3  | AgingSystem LifeStage | YoungAdult / Elder / Ancient phân loại đúng |
| 4  | InjurySystem apply+recover | Minor debuff ATK, recover sau 5 ngày → ATK về baseline |
| 5  | InjurySystem priority | Severe không bị ghi đè bởi Minor |
| 6  | Disciple counters | RecordKill(10+5)=15, RecordDispatch×3=3 |
| 7  | TraitUnlock AND | "realm>=2 AND dispatch>=1" — chưa unlock khi dispatch=0, unlock khi dispatch=1 |
| 8  | QuestChain A→B | Complete A → B tự unlock lên board |
| 9  | CompetitorMarket | ActivityLevel=1.0, 30 ngày → giá LingCao thay đổi |
| 10 | BuildingEfficiency tier | Tier 2 = Tier 1 × 1.10 (±float precision) |

---

## Checklist M6

**Sim / Disciples**
- [x] `Sim/Disciples/Disciple.cs` — +KillCount, +DispatchCount, RecordKill(), RecordDispatch()
- [x] `Sim/Disciples/AgingSystem.cs` — milestone 10 năm, LifeStage, near-death 10%, age efficiency mult
- [x] `Sim/Disciples/InjurySystem.cs` — Minor/Moderate/Severe, stat debuff, recovery, priority ghi đè
- [x] `Sim/Disciples/TraitUnlockSystem.cs` — AND operator, wired kills/dispatch counter

**Sim / Quests**
- [x] `Sim/Quests/QuestChain.cs` — QuestChain (ordered steps) + QuestChainRegistry (event bridge)
- [x] `Sim/Quests/DispatchSystem.cs` — wire InjurySystem, RecordDispatch(), publish QuestCompletedEvent

**Sim / Economy**
- [x] `Sim/Economy/CompetitorMarket.cs` — CompetitorMarket (single) + CompetitorMarketPool (multi)

**LocalMap**
- [x] `LocalMap/Buildings/BuildingEfficiencyTable.cs` — TierBonusPerTier=0.10, ComputeEfficiencyBase()
- [x] `LocalMap/LocalMapManager.cs` — integrate 4 hệ thống M6, tick 14 bước

**Events**
- [x] `Core/Events/SampleEvents.cs` — +8 event M6 mới

**Tests**
- [x] `Tests/Editor/M6SimTest.cs` — 10 tests qua menu Nizhangjie/Run M6 Tests

---

## Pending (M7+)

- [ ] AgingSystem.GetAgeEfficiencyMultiplier → wire vào BuildingController (áp dụng lifecycle penalty)
- [ ] InjurySystem: wired từ CombatEncounter (HP thấp sau combat → injury)
- [ ] QuestChain: phân nhánh A → (B hoặc C) tùy điều kiện / lựa chọn player
- [ ] TraitUnlockSystem: OR operator, scripted condition từ JSON
- [ ] CompetitorMarket: budget hữu hạn, cạnh tranh thật (buy low, sell high)
- [ ] Disciple relationships (thầy-trò, huynh đệ) → loyalty buff/debuff
- [ ] World map layer: nhiều settlement, faction war ảnh hưởng market/quest
- [ ] Disciple death legacy: đệ tử chết để lại di vật / kỹ năng cho thầy
