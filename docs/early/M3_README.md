# M3 — Combat System

**Namespace:** `Nizhangjie.Combat.*`

M3 tách hệ thống combat cũ thành các component rõ ràng, thêm extensible hitshape,
DamageCalculator dùng công thức chuẩn, và StatusEffect pipeline đầy đủ.

---

## Cấu trúc thư mục

```
Combat/
├── Encounter/
│   ├── CombatEncounter.cs      — MonoBehaviour quản lý vòng đời encounter
│   └── CombatGrid.cs           — Grid 2D, BFS pathfinding, occupancy tracking
├── Units/
│   ├── CombatUnit.cs           — Orchestrator (không chứa logic)
│   ├── UnitHealth.cs           — HP, TakeDamage, Heal, Die → fires bus events
│   ├── UnitMovement.cs         — BFS movement, MoveToward, GetReachableTiles
│   └── UnitCombatComponent.cs  — Basic attack qua DamageCalculator
├── Skills/
│   ├── Skill.cs                — Runtime instance: cooldown, cast state, hitbox
│   ├── SkillExecutor.cs        — 4 slots, equip/unequip, UseSkill, TickCast
│   ├── Hitbox.cs               — IHitShape → danh sách CombatUnit bị hit
│   └── HitShapes/
│       ├── IHitShape.cs        — Interface mở rộng được (moddable)
│       ├── RectangleShape.cs   — W×H tile centered at origin
│       ├── LineShape.cs        — Line theo direction, dừng khi gặp terrain
│       ├── ConeShape.cs        — Nón mở rộng theo range + spreadFactor
│       └── CrossShape.cs       — Chữ thập (+) 4 cánh
├── Damage/
│   └── DamageCalculator.cs     — ATK * mult → DEF mitigation → elemental resist → crit
├── StatusEffects/
│   ├── StatusEffect.cs         — Data class: type, duration, magnitude, stack
│   ├── StatusEffectController.cs — Apply, Remove, Tick DoT/HoT mỗi turn
│   └── ElementalReaction.cs   — Phản ứng nguyên tố khi 2 element gặp nhau
└── Terrain/
    └── CombatTerrain.cs        — Obstacle, Hazard, Cover tiles
```

---

## Core concepts

### CombatUnit — hai identity

```csharp
// Faction = world-level (karma, reputation, meta)
// Team    = encounter-level (damage check, target selection)
var unit = new CombatUnit(
    unitId:      "disciple:elder_sun",
    faction:     Faction.Player,
    stats:       myStats,
    maxHP:       500f,
    moveRange:   4,
    attackRange: 1
);

// UnitId "disciple:*" → CombatEncounter bridge sang DiscipleDiedEvent (GlobalBus)
```

### CombatEncounter lifecycle

```csharp
// 1. Tạo GameObject + component
var go = new GameObject("CombatEncounter");
var enc = go.AddComponent<CombatEncounter>();

// 2. (optional) Override grid size trước Begin
enc.Configure(gridWidth: 16, gridHeight: 10);

// 3. Đăng ký units — PlaceUnit tự động, Initialize inject bus/grid
enc.RegisterUnit(unitA, startTile: new Vector2Int(1, 4));
enc.RegisterUnit(unitB, startTile: new Vector2Int(14, 4));

unitA.Team = Team.A;
unitB.Team = Team.B;

// 4. Đăng ký result handler
enc.OnEncounterEnded += winner => Debug.Log($"Winner: {winner}");

// 5. Bắt đầu
enc.Begin();

// 6. Cleanup khi xong
Destroy(go);
```

### Skills — equip và sử dụng

```csharp
// Load definition từ DataRegistry
var fireBallDef = DataRegistry<SkillDefinition>.Get("skill_fireball");
// fireball JSON: hitShapeType="cone", hitShapeRange=3, hitShapeWidth=10 (= spread 1.0)

// Equip vào slot 1
unit.Skills.EquipSkill(SkillExecutor.SlotActive1, fireBallDef);

// Sử dụng (instant cast nếu castTime=0, hoặc cần TickCast)
bool started = unit.Skills.UseSkill(SkillExecutor.SlotActive1, targetTile, encounter);

// Nếu có castTime > 0, gọi mỗi frame:
bool fired = unit.Skills.TickCast(Time.deltaTime, targetTile, encounter);

// Interrupt cast
unit.Skills.InterruptCast();
```

### DamageCalculator — công thức

```
rawDamage  = ATK × damageMultiplier
effectDEF  = DEF × (1 - ArmorPenetration)
mitigation = effectDEF / (effectDEF + 100)        [K = 100]
afterMit   = rawDamage × (1 - mitigation)
afterResist = afterMit × (1 - elementalResistance)
finalDamage = isCrit ? afterResist × CritDMG : afterResist   [minimum 1]
```

### Elemental Reactions

| Incoming + Existing | Reaction | Hiệu ứng |
|---|---|---|
| Fire + Freeze | Melt | +10% MaxHP bonus dmg, xóa cả hai |
| Fire + Poison | Combust | +5% MaxHP bonus dmg, xóa poison |
| Lightning + Freeze | Shatter | Stun 1 turn, xóa freeze |
| Ice + Wet | Freeze | Freeze 2 turn |
| Fire + Wood Mark | Ignite | Burn DoT mạnh 3 turn |
| Earth + Electrify | Magnetize | DEF -30% 2 turn |

### IHitShape — thêm shape tùy ý

```csharp
// Implement interface để thêm shape mới (modder-friendly)
public class SpiralShape : IHitShape
{
    public List<Vector2Int> GetAffectedTiles(Vector2Int origin, Vector2Int direction, CombatGrid grid)
    {
        // logic xoắn ốc tùy ý
    }
}

// Dùng trực tiếp với Hitbox
var hitbox = new Hitbox(new SpiralShape());
var targets = hitbox.GetTargets(origin, direction, encounter, attackerTeam);
```

---

## Events (CombatEventBus)

| Event | When | Key fields |
|---|---|---|
| `SkillCastEvent` | Khi bắt đầu cast | CasterId, SkillId, TargetTile |
| `DamageDealtEvent` | Sau khi TakeDamage | AttackerId, VictimId, Damage, DamageType, IsCritical |
| `CombatUnitDiedEvent` | Khi unit HP ≤ 0 | UnitId, Team, KillerUnitId |
| `StatusAppliedEvent` | Khi apply buff/debuff | UnitId, EffectType, Duration |
| `StatusExpiredEvent` | Khi effect hết turn | UnitId, EffectType |
| `ElementalReactionEvent` | Khi reaction xảy ra | VictimId, ReactionName, BonusDamage |
| `UnitMovedEvent` | Mỗi tile di chuyển | UnitId, From, To |
| `UnitTurnStartedEvent` | Đầu turn của unit | UnitId, Team, Round |

Bridge: `CombatUnitDiedEvent` (combat bus) → `DiscipleDiedEvent` (global bus)
nếu `UnitId.StartsWith("disciple:")`.

---

## Checklist M3

- [x] `CombatGrid` — BFS pathfinding, occupancy, terrain block
- [x] `CombatEncounter` — lifecycle, win condition, bus bridge
- [x] `CombatUnit` — orchestrator, 2 identity (Faction + Team)
- [x] `UnitHealth` — HP, TakeDamage, Heal, Die
- [x] `UnitMovement` — MoveTo, MoveToward, GetReachableTiles
- [x] `UnitCombatComponent` — basic attack qua DamageCalculator
- [x] `IHitShape` + 4 shapes (Rectangle, Line, Cone, Cross)
- [x] `Hitbox` — shape → targets + factory method
- [x] `Skill` — cooldown, cast state, hitbox
- [x] `SkillExecutor` — 4 slots, passive modifier, UseSkill, TickCast
- [x] `DamageCalculator` — ATK/DEF/resist/crit/lifesteal
- [x] `StatusEffect` + `StatusEffectController` — apply, tick, dispel
- [x] `ElementalReaction` — 6 reaction pairs
- [x] `CombatTerrain` — obstacle, hazard, cover
- [x] Combat events đầy đủ trong `SampleEvents.cs`

---

## Pending (M4+)

- [ ] Turn order system (SPD-based initiative queue)
- [ ] AI behavior (BehaviorTree hoặc utility AI)
- [ ] CombatUnit factory từ Disciple (link Stats + Equipment)
- [ ] DamageFormula load từ JSON (DataRegistry)
- [ ] Terrain definition load từ JSON
- [ ] Line-of-sight check cho ranged skills
- [ ] Elevation system
- [ ] Visual layer (tween, VFX, UI HUD) — hoàn toàn tách khỏi logic
