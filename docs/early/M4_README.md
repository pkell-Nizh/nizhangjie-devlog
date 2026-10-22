# M4 — Sim Skeleton

**Namespace:** `Nizhangjie.LocalMap.*` + `Nizhangjie.Sim.*`

M4 xây dựng khung quản lý tông môn: local map grid, building system, disciple management, economy cơ bản, và quest board.

---

## Cấu trúc thư mục

```
LocalMap/
├── Grid/
│   ├── Tile.cs              — Ô trên map (type, passable, occupant, zone, lingQi)
│   └── GridSystem.cs        — 2D grid, PlaceBuilding, FillRect, zone assignment
├── Zoning/
│   ├── ZoneType.cs          — Enum: Cultivation, Farming, Mining, Living, ...
│   └── Zone.cs              — Vùng player đánh dấu (tập hợp tiles + worker capacity)
├── Pathfinding/
│   └── LocalPathfinder.cs   — A* với tile cost (Road=0.5, Forest=2, Swamp=3)
└── Buildings/
    ├── Building.cs           — Runtime instance: tier, footprint, workers, state
    ├── BuildingController.cs — Tick production, map output → SectResources
    └── WorkAssignment.cs     — Manual assign + standing order

Sim/
├── Sect/
│   ├── Sect.cs              — Aggregate: tên, rank, disciple roster, reputation + unlock tiers
│   └── SectResources.cs     — 15 resources: Cơ bản, Hiếm, Quái thú, Tiêu hao
├── Disciples/
│   ├── Disciple.cs          — CharacterStats + traits + equipment + loyalty (no Desire)
│   ├── DiscipleRegistry.cs  — Central registry, bridge DiscipleDied + DiscipleLeft events
│   ├── Recruitment.cs       — Generate candidate, recruit với cost
│   ├── SalarySystem.cs      — Trả lương tháng, xử lý bỏ đi + trộm đồ
│   ├── ExpectationSystem.cs — (legacy placeholder)
│   └── Traits/
│       └── Trait.cs         — Runtime wrapper TraitDefinition, apply/remove modifier
├── Economy/
│   ├── Market.cs            — Fixed price buy/sell tại settlement
│   └── Shop.cs              — Player shop: passive income + NPC customer spawn
└── Quests/
    ├── Quest.cs             — State machine + RequiresCombat / Difficulty / DispatchDays
    ├── QuestRegistry.cs     — Central registry, advance objective, daily tick
    ├── QuestBoard.cs        — Per-settlement board, refresh, player accept
    └── DispatchSystem.cs    — Phái đội: countdown → RNG vs combat power → auto-resolve
```

---

## Core concepts

### Resource system (15 loại, GDD v2)

```csharp
// [Cơ bản]
ResourceType.LingThach    // linh thạch — tiền tệ chính
ResourceType.LingCao      // linh thảo phổ thông (farming input)
ResourceType.KhoangThach  // khoáng thạch (mining output)

// [Hiếm / Đặc thù]
ResourceType.GoHiem       // gỗ hiếm (building Tier 4-5)
ResourceType.ThaoDuocHiem // thảo dược hiếm (đan dược cấp cao)
ResourceType.DaoQua       // đạo quả (công thức cấp cao nhất)

// [Nguyên liệu quái thú]
ResourceType.YeuDan       // yêu đan (core của yêu thú)
ResourceType.DaQuai       // da quái thú
ResourceType.RangVuot     // răng/vuốt

// [Tiêu hao / Phát triển]
ResourceType.Dan          // đan dược tổng hợp
ResourceType.Bua          // bùa chú
ResourceType.KhiCu        // khí cụ / vũ khí chế tạo
ResourceType.CongPhap     // công pháp (skill book)
ResourceType.BiTich       // bí tịch (EXP nghề phụ)
ResourceType.LinhKhi      // linh khí tích trữ

// Thao tác
sect.Resources.Add("ling_thach", 100f);
sect.Resources.Spend("ling_cao", 10f);       // false nếu không đủ
sect.Resources.SpendMultiple(costDict);      // atomic, fail nếu thiếu bất kỳ cái nào
```

### Salary system (thay Desire)

```csharp
// SalarySystem.cs — trả lương mỗi 30 ngày
var salary = new SalarySystem(registry, globalBus);

// Tính lương một đệ tử (realm=1 → 30 LingThach/tháng base)
float monthly = salary.GetMonthlySalary(disciple);
// Nếu Trait Tham Lam (salaryMultiplier=1.25) → monthly *= 1.25

// Daily tick → mỗi 30 ngày tự trả lương
salary.OnDayPassed(sect.Resources);

// Khi không đủ tiền:
//   disciple.Loyalty -= 20
//   Nếu Loyalty = 0 → DiscipleLeftSectEvent (15% chance DiscipleTheftEvent trước)
```

### Disciple lifecycle

```csharp
// 1. Generate candidate từ UnitDefinition JSON
var candidate = Recruitment.GenerateCandidate(unitDef, "disciple_001", "Trần Lập", Faction.Player, traitPool);

// 2. Recruit
var recruitment = new Recruitment { BaseRecruitCost = 150f };
bool ok = recruitment.Recruit(candidate, sect, registry);
// ok = false nếu sect đầy hoặc không đủ linh thạch

// 3. Daily tick (aging, loyalty drift nhẹ)
registry.OnDayPassed(1f / 365f);
salary.OnDayPassed(sect.Resources);    // trả lương mỗi 30 ngày

// 4. Death → DiscipleDiedEvent → DiscipleRegistry.OnDiscipleDied() → Disciple.Die()
// 5. Leave → DiscipleLeftSectEvent → DiscipleRegistry.OnDiscipleLeft() → Unregister()
```

### Building production flow

```csharp
// Setup
var controller = new BuildingController();
controller.SetResources(sect.Resources);
controller.RegisterBuilding(spiritFieldBuilding);

// Manual assign
var workAssign = new WorkAssignment();
workAssign.Assign("disciple_001", "building_001", spiritFieldBuilding);

// Standing order
workAssign.SetStandingOrder("disciple_002", "building_001");

// Mỗi ngày tick production
var efficiencies = new Dictionary<string, float>
{
    ["disciple_001"] = disciple.GetWorkEfficiency()
};
controller.Tick(tickDuration: 1f, efficiencies);
// → tích luỹ output → drain vào SectResources khi đủ 1 unit
```

### Quest + Dispatch flow

```csharp
// Tạo quest Dispatch (farming)
var quest = new Quest("q_001", "Trừ 20 yêu thú nhỏ", "...", QuestTypes.MonsterHunt, "settlement_north")
    .AddObjective(new QuestObjective("Tiêu diệt yêu thú", 20f))
    .SetDeadline(14)
    .SetDifficulty(30f)         // dễ → Dispatch OK
    .SetDispatchDays(5)         // 5 ngày đi về
    .AddReward(ResourceType.LingThach, 500f)
    .AddReward(ResourceType.YeuDan, 5f);
// quest.RequiresCombat = false (default)

// Tạo quest Combat (dungeon boss)
var bossQuest = new Quest("q_boss", "Chinh phục Huyết Nguyệt Động", "...", QuestTypes.Dungeon, "settlement_north")
    .SetRequiresCombat(true)   // mở CombatEncounter thật
    .SetDifficulty(80f);

// Add + player accept
questRegistry.Register(quest);
board.AddQuest(quest);
board.AcceptQuest("q_001");

// Dispatch: phái 2 đệ tử đi
var dispatch = new DispatchSystem(discipleRegistry, questRegistry, globalBus);
dispatch.StartDispatch(quest, new List<string>{ "disciple_001", "disciple_002" }, sect.Resources);

// Daily tick — tất cả hệ thống
questRegistry.OnDayPassed(sect.Resources);
dispatch.OnDayPassed();   // khi hết DispatchDays → auto-resolve + publish DispatchResultEvent
```

### Reputation & unlock tier

```csharp
// Danh tiếng tông môn tại settlement ảnh hưởng:
//   shop NPC spawn rate, candidate rarity, quest tier, building tier cap

sect.AddPrestige(10f);                    // tăng SectPrestige (0-100)
sect.ModifyFactionRep("settlement_north", 15f);  // tăng rep với faction cụ thể

int prestige = sect.PrestigeTier;         // 0-4 (Vô danh → Huyền thoại)
int maxTier  = sect.MaxBuildingTier;      // tier building tối đa được phép xây
int maxRar   = sect.MaxCandidateRarity;   // candidate rarity tối đa (0=Thường, 3=Thiên tài)
int maxQuest = sect.MaxQuestTier;         // tier quest board (1/2/3)

// Shop sync reputation:
shop.SettlementReputation = sect.GetFactionRep("settlement_north") + 50f;  // map -100..100 → 0..100 (rough)
```

### Work efficiency

```
WorkEfficiency = f(CoreStats, loyalty) — KHÔNG phụ thuộc equipment/combat stat
baseEff        = (Agility + Foundation) / 200
loyaltyMod     = Loyalty / 100
workEff        = baseEff × loyaltyMod   [0-1]
```

Per-building type efficiency (M5+): Linh Điền dùng Foundation, Khai Khoáng dùng Force, Luyện Đan dùng Comprehension + Agility.

---

## LocalMap pathfinding — tile costs

| TileType | MoveCost | Buildable |
|---|---|---|
| Road | 0.5 | No |
| Grass / Stone / Sacred | 1.0 | Yes |
| Forest | 2.0 | Chặt trước |
| Swamp | 3.0 | No |
| Water / Mountain | ∞ (impassable) | No |

---

## Checklist M4

**LocalMap**
- [x] `LocalMap/Grid/Tile.cs` — tile data (type, passable, occupant, zone, lingQi)
- [x] `LocalMap/Grid/GridSystem.cs` — 2D grid, PlaceBuilding, zone, lingQi
- [x] `LocalMap/Zoning/ZoneType.cs` + `Zone.cs`
- [x] `LocalMap/Pathfinding/LocalPathfinder.cs` — A* với tile cost
- [x] `LocalMap/Buildings/Building.cs` — tier, footprint, workers, state, efficiency
- [x] `LocalMap/Buildings/BuildingController.cs` — production tick → SectResources
- [x] `LocalMap/Buildings/WorkAssignment.cs` — manual assign + standing order

**Sim / Sect**
- [x] `Sim/Sect/Sect.cs` — aggregate, roster, reputation, rank, unlock tier helpers
- [x] `Sim/Sect/SectResources.cs` — 15 resources (Cơ bản/Hiếm/Quái thú/Tiêu hao), SpendMultiple

**Sim / Disciples**
- [x] `Sim/Disciples/Disciple.cs` — CharacterStats, traits, equipment, loyalty (Desire removed)
- [x] `Sim/Disciples/DiscipleRegistry.cs` — CRUD, bridge DiscipleDied + DiscipleLeft, daily tick
- [x] `Sim/Disciples/Recruitment.cs` — GenerateCandidate, per-stat roll, trait roll
- [x] `Sim/Disciples/Traits/Trait.cs` — Apply/Remove modifier, source key
- [x] `Sim/Disciples/SalarySystem.cs` — trả lương tháng, bỏ đi, trộm đồ
- [x] `Data/Definitions/TraitDefinition.cs` — thêm `salaryMultiplier` field

**Sim / Economy**
- [x] `Sim/Economy/Market.cs` — fixed price buy/sell
- [x] `Sim/Economy/Shop.cs` — passive income + NPC customer spawn (reputation-based)

**Sim / Quests**
- [x] `Sim/Quests/Quest.cs` — state machine + RequiresCombat / Difficulty / DispatchDays / QuestTypes
- [x] `Sim/Quests/QuestRegistry.cs` — CRUD, AdvanceObjective, daily tick
- [x] `Sim/Quests/QuestBoard.cs` — per-settlement, refresh, player accept
- [x] `Sim/Quests/DispatchSystem.cs` — phái đội, countdown, RNG resolve, injury/permadeath

**Events**
- [x] `Core/Events/SampleEvents.cs` — thêm DiscipleLeftSectEvent, DiscipleTheftEvent, DispatchStartedEvent, DispatchResultEvent

**Tests**
- [x] `Tests/Editor/M3CombatTest.cs` — 10 tests qua menu Nizhangjie/Run M3 Tests

---

## Pending (M5+)

- [ ] `LocalMapManager.cs` — MonoBehaviour orchestrator (tick tất cả hệ thống mỗi ngày)
- [ ] Building placement UI (drag footprint lên grid)
- [ ] Zone painter UI (paint tiles vào zone)
- [ ] Per-building-type work efficiency (Linh Điền → Foundation, Khai Khoáng → Force)
- [ ] Disciple detail panel (show traits, stats, desire bar)
- [ ] Storyteller quest generation (M5)
- [ ] Market price dynamics (supply/demand, M5)
- [ ] Shop competitor NPC (M5)
- [ ] Sect rank promotion requirements (resource + disciple count + prestige threshold)
