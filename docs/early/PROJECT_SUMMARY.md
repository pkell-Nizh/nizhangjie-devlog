# Nizhangjie (Nhĩ Chưởng Giới) — Project Summary

**Engine:** Unity 6 LTS · URP 2D Renderer · C# 9  
**Namespace:** `Nizhangjie.*`  
**Architecture:** Plain C# classes (không MonoBehaviour trừ khi cần), JSON as source of truth, ServiceLocator, EventBus

---

## Triết lý thiết kế

- **JSON-first**: tất cả data (unit, skill, trait, building...) define trong JSON, không ScriptableObject
- **No singletons**: dùng ServiceLocator thay `Instance`
- **EventBus**: mọi thông tin cross-system đi qua bus, không direct reference
- **Extensible**: string keys thay enum cho resource/quest type → moddable
- **Testable**: logic trong plain C# class, không phụ thuộc Unity lifecycle

---

## M1 — EventBus

| File | Mô tả |
|---|---|
| `Core/Events/IEventBus.cs` | Interface: Subscribe/Unsubscribe/Publish/Interceptor |
| `Core/Events/EventBus.cs` | Implementation: priority-sorted, snapshot iteration |
| `Core/Events/ScopedBuses.cs` | GlobalEventBus · CombatEventBus · LocalMapEventBus |
| `Core/ServiceLocator.cs` | Dependency injection, không MonoBehaviour |
| `Core/Events/SampleEvents.cs` | Tất cả event classes (xem bảng cuối) |

**Pattern chính:**
```csharp
// Subscribe
globalBus.Subscribe<DiscipleDiedEvent>(OnDiscipleDied, priority: 10);

// Publish
globalBus.Publish(new DiscipleDiedEvent { DiscipleId = id, CauseTag = "age" });

// Bridge: Combat → Global
combatBus.Subscribe<CombatUnitDiedEvent>(e => {
    if (e.UnitId.StartsWith("disciple:"))
        globalBus.Publish(new DiscipleDiedEvent { DiscipleId = e.UnitId[9..] });
});
```

---

## M2 — Data Pipeline + Stats + Equipment

### Stats system
| File | Mô tả |
|---|---|
| `Stats/StatType.cs` | Enum 20+ stats: Aptitude/Force/Foundation/Agility/Comprehension/Charisma + combat |
| `Stats/StatModifier.cs` | Struct: Flat → PercentAdd (additive sum) → PercentMul (compound) |
| `Stats/CharacterStats.cs` | Dictionary-based, lazy compute, RemoveModifiersFromSource(key) |

**Modifier stack:**
```
finalStat = (base + Σflat) × (1 + ΣpercentAdd) × Π(1 + percentMul_i)
```

### Data pipeline
| File | Mô tả |
|---|---|
| `Data/IDefinition.cs` | Interface: `string Id` |
| `Data/Loaders/JsonDataLoader.cs` | Load POCO từ JSON, JsonUtility |
| `Data/Loaders/DataRegistry<T>.cs` | Generic lookup by ID |
| `Data/Definitions/UnitDefinition.cs` | Template nhân vật: stat ranges, realm, traits, isKeyUnit |
| `Data/Definitions/SkillDefinition.cs` | Skill data: type, hitShape, cooldown, cast time |
| `Data/Definitions/TraitDefinition.cs` | Trait: modifiers[], workEfficiencyBonus, **salaryMultiplier** |
| `Data/Definitions/BuildingDefinition.cs` | Building: tier cap, output resource, worker capacity |
| `Data/Definitions/ItemDefinition.cs` | Item: slot, modifiers |
| `Data/Definitions/QuestDefinition.cs` | Quest template |
| `Data/Definitions/EnemyDefinition.cs` | Enemy archetype |

### Equipment
| File | Mô tả |
|---|---|
| `Equipment/EquipmentItem.cs` | Wrap ItemDefinition, apply/remove modifier với source key |
| `Equipment/Inventory.cs` | 9 slots (KeyUnit có thêm 2 Artifact slots), auto-apply vào CharacterStats |

---

## M3 — Combat System

### Grid & Encounter
| File | Mô tả |
|---|---|
| `Combat/Encounter/CombatGrid.cs` | 2D grid: PlaceUnit, FindPath (BFS), ManhattanDistance |
| `Combat/Encounter/CombatEncounter.cs` | Orchestrator: RegisterUnit, turn order, bridge Died event |

**Bridge pattern:**
```csharp
// CombatEncounter: khi unit chết trong combat
if (unitId.StartsWith("disciple:"))
    _globalBus.Publish(new DiscipleDiedEvent {
        DiscipleId = unitId[9..],
        KillerId   = killerUnitId,
        CauseTag   = "combat"
    });
```

**RegisterUnit ordering (fix bug):** PlaceUnit trước, sau đó Initialize — để unit đọc được position từ grid.

### Units & Components
| File | Mô tả |
|---|---|
| `Combat/Units/CombatUnit.cs` | 5 components. Faction (world) + Team (encounter). Initialize inject bus/grid |
| `Combat/Units/UnitHealth.cs` | HP, TakeDamage, Die, lifesteal. Publish DamageDealtEvent + CombatUnitDiedEvent |
| `Combat/Units/UnitMovement.cs` | Move với SPD budget, publish UnitMovedEvent |
| `Combat/Units/UnitCombatComponent.cs` | CanAttack (range/team), PerformBasicAttack → DamageCalculator |

### Skills & Hit Shapes
| File | Mô tả |
|---|---|
| `Combat/Skills/HitShapes/IHitShape.cs` | Interface: GetAffectedTiles(origin, direction, grid) |
| `Combat/Skills/HitShapes/RectangleShape.cs` | halfWidth × halfHeight, axis-aligned, origin IN result |
| `Combat/Skills/HitShapes/LineShape.cs` | Line theo direction, stops at obstacle, origin NOT in result |
| `Combat/Skills/HitShapes/ConeShape.cs` | Widening: at depth n, lateral ≤ n × spreadFactor |
| `Combat/Skills/HitShapes/CrossShape.cs` | + pattern 4 hướng, mỗi arm stops at obstacle |
| `Combat/Skills/Hitbox.cs` | Eval IHitShape → filter List<CombatUnit> theo team. Factory FromDefinition() |
| `Combat/Skills/Skill.cs` | Runtime: cooldown, cast time, BeginCast/TickCast/CancelCast |
| `Combat/Skills/SkillExecutor.cs` | 4 slots (Normal/Active1/Active2/Passive). Passive auto-apply modifier |

### Damage, Status, Elemental
| File | Mô tả |
|---|---|
| `Combat/Damage/DamageCalculator.cs` | Static: ATK×mult → DEF/(DEF+100) → elemental resist → crit |
| `Combat/StatusEffects/StatusEffect.cs` | Type/duration/magnitude/stack. Refresh() stack++, Tick() turn-- |
| `Combat/StatusEffects/StatusEffectController.cs` | Apply/Remove/Dispel, OnTurnStart() tick DoT/HoT |
| `Combat/StatusEffects/ElementalReaction.cs` | 6 reactions: Melt/Combust/Shatter/Freeze/Ignite/Magnetize |
| `Combat/Terrain/CombatTerrain.cs` | TerrainType enum, hazard tick mỗi turn |

**Damage formula:**
```
rawDmg     = ATK × damageMultiplier
mitigation = effectiveDEF / (effectiveDEF + 100)   [K=100]
afterMit   = rawDmg × (1 - mitigation)
afterRes   = afterMit × (1 - elementalResist)
finalDmg   = isCrit ? afterRes × CritDMG : afterRes   [min 1]
```

**6 Elemental reactions:**
| Reaction | Elements | Effect |
|---|---|---|
| Melt | Fire + Ice | +10% MaxHP bonus dmg |
| Combust | Fire + Poison | +5% MaxHP bonus dmg |
| Shatter | Ice + Lightning | Stun 1 turn |
| Freeze | Ice + Water | Freeze 2 turns |
| Ignite | Fire + WoodMark | Burn DoT 3 turns (mạnh) |
| Magnetize | Earth + Lightning | DEF -30% 2 turns |

---

## M4 — Sim Skeleton

### LocalMap
| File | Mô tả |
|---|---|
| `LocalMap/Grid/Tile.cs` | TileType (8 loại), passable, occupant, zoneId, lingQiModifier |
| `LocalMap/Grid/GridSystem.cs` | 2D Tile[,], PlaceBuilding validate footprint, GetLingQiInRect |
| `LocalMap/Zoning/ZoneType.cs` | Enum 10 loại: Cultivation/Farming/Mining/Crafting/Living/Defense/Storage/Sacred/Restricted |
| `LocalMap/Zoning/Zone.cs` | HashSet<Vector2Int>, worker capacity, AddTile sync GridSystem |
| `LocalMap/Pathfinding/LocalPathfinder.cs` | A* với SortedSet<AStarNode>, tile costs |
| `LocalMap/Buildings/Building.cs` | Tier 1-5, BuildingState, MaxWorkers, GetBaseEfficiency() |
| `LocalMap/Buildings/WorkAssignment.cs` | Manual assign + standing order, TryAutoAssign |
| `LocalMap/Buildings/BuildingController.cs` | Tick production per game-hour → SectResources |

**Tile costs (A\*):**
| TileType | MoveCost | Buildable |
|---|---|---|
| Road | 0.5 | No |
| Grass / Stone / Sacred | 1.0 | Yes |
| Forest | 2.0 | Chặt trước |
| Swamp | 3.0 | No |
| Water / Mountain | ∞ | No |

### Sim / Sect
| File | Mô tả |
|---|---|
| `Sim/Sect/Sect.cs` | Roster, SectRank 0-4, SectPrestige 0-100, factionRep dict, unlock tier helpers |
| `Sim/Sect/SectResources.cs` | 15 resources, Add/Spend/SpendMultiple (atomic) |

**15 Resource types:**
| Nhóm | Keys |
|---|---|
| Cơ bản | LingThach · LingCao · KhoangThach |
| Hiếm | GoHiem · ThaoDuocHiem · DaoQua |
| Quái thú | YeuDan · DaQuai · RangVuot |
| Tiêu hao | Dan · Bua · KhiCu · CongPhap · BiTich · LinhKhi |

**Reputation unlock tiers:**
| Prestige | Tier | MaxBuilding | MaxCandidate | MaxQuest |
|---|---|---|---|---|
| 0-19 | 0 Vô danh | 1 | Thường nhân | 1 |
| 20-39 | 1 Mới nổi | 2 | Tài năng | 1 |
| 40-59 | 2 Có tiếng | 3 | Tài năng | 2 |
| 60-79 | 3 Nổi tiếng | 4 | Kỳ tài | 3 |
| 80-100 | 4 Huyền thoại | 5 | Thiên tài | 3 |

### Sim / Disciples
| File | Mô tả |
|---|---|
| `Sim/Disciples/Disciple.cs` | Stats + Traits + Equipment + Loyalty. GetWorkEfficiency(). No Desire |
| `Sim/Disciples/DiscipleRegistry.cs` | CRUD, bridge DiscipleDied + DiscipleLeft, Alive(), AllDisciples |
| `Sim/Disciples/Recruitment.cs` | GenerateCandidate, per-stat roll, Fisher-Yates trait shuffle |
| `Sim/Disciples/Traits/Trait.cs` | Apply/Remove modifier, source key "trait:{id}" |
| `Sim/Disciples/SalarySystem.cs` | Trả lương mỗi 30 ngày, bỏ đi, trộm đồ |

**Work efficiency:**
```
workEff = (Agility + Foundation) / 200 × Loyalty / 100   [0-1]
```

**Salary system:**
```
salary  = RealmBaseSalary[realm] × Π(trait.salaryMultiplier)
Realm:    0→10  1→30  2→80  3→200  4→500  5+→1000  (LingThach/tháng)

Thiếu lương → Loyalty -= 20/tháng
Loyalty = 0 → DiscipleLeftSectEvent
           + 15% → DiscipleTheftEvent (trộm 5-20% 1 resource ngẫu nhiên)
```

### Sim / Economy
| File | Mô tả |
|---|---|
| `Sim/Economy/Market.cs` | Buy/sell fixed price, spread sell ≈ 60-80% |
| `Sim/Economy/Shop.cs` | Passive daily sale + NPC customer spawn (reputation-based) |

**NPC customer spawn:**
```
spawnChance = SettlementReputation / 200
  → rep=50  = 25% / ngày
  → rep=100 = 50% / ngày
NPC mua 1-5 units mặt hàng ngẫu nhiên, đúng giá player set
```

### Sim / Quests
| File | Mô tả |
|---|---|
| `Sim/Quests/Quest.cs` | State machine + RequiresCombat / Difficulty / DispatchDays / QuestTypes |
| `Sim/Quests/QuestRegistry.cs` | CRUD, AdvanceObjective, OnDayPassed tick + auto-complete |
| `Sim/Quests/QuestBoard.cs` | Per-settlement, max 5 quests, refresh 7 ngày |
| `Sim/Quests/DispatchSystem.cs` | Phái đội: countdown → RNG → auto-resolve |

**Quest types (QuestTypes static class):**
`Trading` · `MonsterHunt` · `Punitive` · `Bandit` · `Dungeon` · `Craft` · `Delivery` · `Exploration`

**Dispatch formula:**
```
power   = Σ (ATK + Foundation×0.5) × loyalty/100
chance  = power / (power + difficulty×10)   [clamp 0.05–0.95]

roll < chance×0.8  → Success        (full reward + TryComplete)
roll < chance      → PartialSuccess  (50% reward, loyalty -10 1 đệ tử)
roll ≥ chance      → Failure         (no reward, 20% permadeath 1 đệ tử)
```

---

## Tất cả Events

| Event | Bus | Trigger |
|---|---|---|
| `DiscipleDiedEvent` | Global | Chết vì bất kỳ lý do |
| `DiscipleRecruitedEvent` | Global | Tuyển mộ thành công |
| `DiscipleLeftSectEvent` | Global | Bỏ đi (lương, loyalty) |
| `DiscipleTheftEvent` | Global | Trộm đồ trước khi rời |
| `RealmBreakthroughEvent` | Global | Đột phá cảnh giới |
| `DayPassedEvent` | Global | Mỗi ngày in-game |
| `DispatchStartedEvent` | Global | Đội phái xuất phát |
| `DispatchResultEvent` | Global | Dispatch hoàn tất (0=Success/1=Partial/2=Fail) |
| `CombatUnitDiedEvent` | Combat | Chết trong trận |
| `SkillCastEvent` | Combat | Skill bắt đầu cast |
| `DamageDealtEvent` | Combat | Damage áp lên unit |
| `StatusAppliedEvent` | Combat | Status effect áp lên |
| `StatusExpiredEvent` | Combat | Status hết hạn |
| `ElementalReactionEvent` | Combat | Phản ứng nguyên tố |
| `UnitMovedEvent` | Combat | Unit di chuyển (mỗi tile) |
| `UnitTurnStartedEvent` | Combat | Turn bắt đầu |
| `BuildingPlacedEvent` | LocalMap | Building được đặt lên grid |

---

## Tests

**`Tests/Editor/M3CombatTest.cs`** — menu `Nizhangjie/Run M3 Tests`

10 tests không cần Play Mode:
1. CombatGrid PlaceUnit + occupant check
2. CombatGrid BFS FindPath
3. CombatGrid ManhattanDistance
4. DamageCalculator công thức ATK/DEF/crit
5. RectangleShape(1,1) → 9 tiles
6. LineShape(3) → 3 tiles, direction đúng
7. CrossShape(1) → 5 tiles
8. ConeShape(2, 1.0) → ≥3 tiles
9. Hitbox.FromDefinition factory (single/rect/line)
10. StatusEffect stack + tick + expire

---

## Pending (M5+)

- [ ] `LocalMapManager.cs` — MonoBehaviour orchestrator: tick tất cả hệ thống mỗi ngày
- [ ] Building placement UI (drag footprint lên grid)
- [ ] Zone painter UI (paint tiles → zone)
- [ ] Per-building-type work efficiency (Linh Điền→Foundation, Khai Khoáng→Force, Luyện Đan→Comprehension)
- [ ] Sect rank promotion (resource threshold + disciple count + prestige)
- [ ] Market price dynamics (supply/demand curve)
- [ ] Storyteller quest generation (procedural quest từ world state)
- [ ] Shop competitor NPC
- [ ] Disciple detail panel UI
- [ ] Combat UI + turn resolution visual
- [ ] HeavenlyTribulation system (đột phá cảnh giới có rủi ro)
- [ ] TraitUnlockSystem (unlock hidden trait qua gameplay condition)
