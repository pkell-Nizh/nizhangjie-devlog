# M9 — Item Equip & Crafting Progression

**Milestone:** Item equip system (EquipmentManager), Crafting XP + Levels (CraftSkillRegistry), Batch crafting.

---

## Cây thư mục mới / thay đổi

```
Assets/Scripts/
├── Sim/Crafting/
│   ├── ItemSlot.cs              ← NEW (M9) — enum Weapon/Armor/Accessory + source key helper
│   ├── EquipmentManager.cs      ← NEW (M9) — quản lý trang bị per-disciple
│   ├── CraftSkill.cs            ← NEW (M9) — XP + Level per disciple per craft type
│   ├── CraftSkillRegistry.cs    ← NEW (M9) — lưu trữ + query toàn bộ CraftSkill
│   ├── CraftingQueue.cs         ← Updated (M9) — EnqueueBatch() + CraftSkill speed bonus
│   ├── AlchemySystem.cs         ← Updated (M9) — SetCraftSkillRegistry, record XP after craft
│   └── ForgeSystem.cs           ← Updated (M9) — SetCraftSkillRegistry, record XP after craft
├── LocalMap/LocalMapManager.cs  ← Updated (M9) — EquipmentManager + CraftSkillRegistry init + wiring
├── Core/Events/SampleEvents.cs  ← Updated (M9) — 3 new events
└── Tests/Editor/M9SimTest.cs    ← NEW (10 bài test)
```

---

## 1. ItemSlot

**File:** `Sim/Crafting/ItemSlot.cs`

| Slot        | Stat bonus                                              |
|-------------|---------------------------------------------------------|
| Weapon      | +ATK (Flat) = `item.EffectiveATK`                      |
| Armor       | +DEF (Flat) = `item.EffectiveDEF` + MaxHP = DEF × 2    |
| Accessory   | +Foundation = DEF×0.5 + Comprehension = ATK×0.3         |

Source key convention: `"equip:{discipleId}:{slot}"` — dùng để unequip sạch modifier.

---

## 2. EquipmentManager

**File:** `Sim/Crafting/EquipmentManager.cs`

Quản lý trang bị của tất cả đệ tử. Mỗi disciple có 3 slot (Weapon/Armor/Accessory).

### API

```csharp
// Trang bị — trả về item cũ ở slot đó (null nếu trống)
ItemInstance old = equipMgr.Equip("disciple_zhao", weaponItem, ItemSlot.Weapon);

// Tháo — trả về item đã tháo
ItemInstance removed = equipMgr.Unequip("disciple_zhao", ItemSlot.Weapon);

// Query
ItemInstance current = equipMgr.GetEquipped("disciple_zhao", ItemSlot.Armor);
IEnumerable<ItemInstance> all = equipMgr.GetAllEquipped("disciple_zhao");

// Tháo tất cả (khi disciple chết / rời sect)
equipMgr.UnequipAll("disciple_zhao");
```

### Stat modifier lifecycle

```
Equip()   → AddModifier(sourceKey)            → stat tăng ngay
Unequip() → RemoveModifiersFromSource(key)    → stat trở về trạng thái không có item
Re-equip  → Unequip() first → then Equip()   → không bị double-count
```

### Ví dụ thực tế

```
Disciple base ATK = 100
Equip sword Normal (BaseATK=50, StatMult=1.0):
  → ATK modifier Flat +50
  → Disciple.Stats.Get(ATK) = 150

Equip sword Masterpiece (BaseATK=50, StatMult=1.75):
  → cũ (Normal) bị tháo → ATK trở về 100
  → Equip mới → modifier Flat +87.5
  → Disciple.Stats.Get(ATK) = 187.5
```

### Sự kiện

| Event                | Khi nào                  |
|----------------------|--------------------------|
| `ItemEquippedEvent`  | Sau Equip() thành công   |
| `ItemUnequippedEvent`| Sau Unequip() thành công |

---

## 3. CraftSkill

**File:** `Sim/Crafting/CraftSkill.cs`

### XP + Level

| Action        | XP gain   |
|---------------|-----------|
| Craft success | +10 XP    |
| Craft failure | +2 XP     |

| Level | XP to next | Tổng XP cần |
|-------|-----------|-------------|
| 1→2   | 50        | 50          |
| 2→3   | 100       | 150         |
| 3→4   | 150       | 300         |
| 4→5   | 200       | 500         |
| 5→6   | 250       | 750         |
| ...   | ...       | ...         |
| 9→10  | 450       | 2250        |

Max Level = 10. Sau khi đạt max, craft không còn tích XP nữa.

### Speed bonus

```
speedMultiplier = 1.0 + (Level − 1) × 0.10
daysRequired    = Ceil(baseDays / speedMultiplier)
```

| Level | Multiplier | Thời gian giảm | Ví dụ (base 10 ngày) |
|-------|-----------|----------------|----------------------|
| 1     | ×1.0      | 0%             | 10 ngày              |
| 3     | ×1.2      | −17%           | 9 ngày               |
| 5     | ×1.4      | −29%           | 8 ngày               |
| 8     | ×1.7      | −41%           | 6 ngày               |
| 10    | ×1.9      | −47%           | 6 ngày               |

---

## 4. CraftSkillRegistry

**File:** `Sim/Crafting/CraftSkillRegistry.cs`

```csharp
// Lấy hoặc tạo skill
CraftSkill skill = craftSkillReg.GetOrCreate("disciple_zhao", "pill");

// Ghi nhận craft (tự publish CraftLevelUpEvent nếu level tăng)
craftSkillReg.RecordCraft("disciple_zhao", "pill", success: true);

// Tính DaysRequired sau khi áp dụng speed bonus
int days = craftSkillReg.GetDaysRequired("disciple_zhao", "pill", baseDays: 10);

// Query
CraftSkill s = craftSkillReg.Get("disciple_zhao", "equipment");     // null nếu chưa tồn tại
IEnumerable<CraftSkill> all = craftSkillReg.GetAllFor("disciple_zhao");
```

### Wiring (tự động trong LocalMapManager)

```
AlchemySystem.SetCraftSkillRegistry(CraftSkillRegistry)
ForgeSystem.SetCraftSkillRegistry(CraftSkillRegistry)
CraftingQueue.SetCraftSkillRegistry(CraftSkillRegistry)
```

- `AlchemySystem.AttemptCraft()` tự gọi `RecordCraft("pill")` sau mỗi craft.
- `ForgeSystem.AttemptCraft()` tự gọi `RecordCraft(recipe.recipeType)` sau mỗi craft.
- `CraftingQueue.Enqueue()` dùng `GetDaysRequired()` để tính DaysRequired giảm.

---

## 5. Batch Crafting

**File:** `Sim/Crafting/CraftingQueue.cs` — `EnqueueBatch()`

```csharp
// Enqueue N lần cùng một recipe vào building
int enqueued = craftingQueue.EnqueueBatch(
    count:       5,
    recipeId:    "pill_qi_gather",
    discipleId:  "disciple_zhao",
    buildingId:  "building_alchemy_01",
    resources:   sect.Resources,
    primaryStat: StatType.Comprehension
);
// Trả về số job thực sự được enqueue (dừng khi thiếu nguyên liệu)
```

Mỗi job là một `CraftJob` độc lập:
- Ingredients tiêu ngay khi enqueue từng job.
- Nếu resources không đủ giữa chừng → dừng, không tạo thêm.
- Tất cả jobs chờ theo FIFO trong queue của building.

---

## 6. LocalMapManager — Wiring M9

```csharp
// Init (trong InitSystems())
EquipmentManager   = new EquipmentManager(DiscipleRegistry, _globalBus);
CraftSkillRegistry = new CraftSkillRegistry(_globalBus);

// Wire vào crafting systems
AlchemySystem.SetCraftSkillRegistry(CraftSkillRegistry);
ForgeSystem.SetCraftSkillRegistry(CraftSkillRegistry);
CraftingQueue.SetCraftSkillRegistry(CraftSkillRegistry);
```

M9 không cần thêm tick step — EquipmentManager và CraftSkillRegistry hoạt động on-demand (không cần daily update).

---

## 7. M9 Events mới (3 sự kiện)

| Event                | Fields chính                                                          |
|----------------------|-----------------------------------------------------------------------|
| `ItemEquippedEvent`  | DiscipleId, ItemId, OutputId, Slot, QualityTier                      |
| `ItemUnequippedEvent`| DiscipleId, ItemId, OutputId, Slot                                   |
| `CraftLevelUpEvent`  | DiscipleId, CraftType, NewLevel, TotalXP                             |

---

## 8. Test Suite (M9SimTest)

**File:** `Tests/Editor/M9SimTest.cs`  
**Menu:** `Nizhangjie/Run M9 Tests`

| # | Tên test                                              | Kiểm tra                                               |
|---|-------------------------------------------------------|--------------------------------------------------------|
| 1 | `EquipmentManager_EquipWeapon_ATKIncreasedByEffectiveATK` | ATK tăng đúng = EffectiveATK của weapon             |
| 2 | `EquipmentManager_UnequipWeapon_ATKRestored`          | ATK về base sau khi tháo                               |
| 3 | `EquipmentManager_EquipArmor_DEFAndMaxHPIncreased`    | DEF + MaxHP đều tăng đúng                              |
| 4 | `EquipmentManager_EquipAccessory_CompAndFoundationIncreased` | Foundation + Comprehension tăng                  |
| 5 | `EquipmentManager_EquipSameSlot_OldItemUnequipped`    | Equip slot bận → item cũ tháo, ATK tính lại đúng      |
| 6 | `CraftSkill_InitialState_Level1_XP0`                  | Level = 1, XP = 0 khi khởi tạo                        |
| 7 | `CraftSkill_RecordCraft_XPAccumulates`                | Success +10 XP, Failure +2 XP                          |
| 8 | `CraftSkill_LevelUp_CraftLevelUpEvent_Fired`          | Đủ 50 XP → Level 2, CraftLevelUpEvent fired            |
| 9 | `CraftSkill_Level5_ApplySpeedBonus_ReducesDays`       | Level 5, base 10 ngày → 8 ngày (×1.4)                 |
|10 | `CraftingQueue_EnqueueBatch_StopsWhenIngredientsRanOut` | Batch 5 với 3 nguyên liệu → enqueued = 3            |

---

## 9. M9 Checklist

- [x] `ItemSlot` enum + `ItemSlotConfig` source key helper
- [x] `EquipmentManager` — equip, unequip, GetEquipped, GetAllEquipped, UnequipAll
- [x] Stat mapping: Weapon→ATK, Armor→DEF+MaxHP, Accessory→Foundation+Comp
- [x] `CraftSkill` — XP, Level, LevelProgress, RecordCraft, ApplySpeedBonus
- [x] `CraftSkillRegistry` — GetOrCreate, RecordCraft, GetDaysRequired, Get, GetAllFor
- [x] `CraftingQueue` — `EnqueueBatch()`, `SetCraftSkillRegistry()`, speed bonus applied at Enqueue
- [x] `AlchemySystem` — `SetCraftSkillRegistry()`, record XP after AttemptCraft
- [x] `ForgeSystem` — `SetCraftSkillRegistry()`, record XP after AttemptCraft
- [x] `LocalMapManager` — M9 systems init + wiring
- [x] `SampleEvents.cs` — `ItemEquippedEvent`, `ItemUnequippedEvent`, `CraftLevelUpEvent`
- [x] `M9SimTest.cs` — 10 bài test
- [x] `M9_README.md` — tài liệu này

---

## M10+ Pending

- **Shared recipe (thầy dạy trò)** — master-student transfer 1 recipe / 10 ngày
- **Enchantment slot** — thêm modifier đặc biệt vào item (drop hoặc mua)
- **Hidden recipes** — yêu cầu vật phẩm đặc biệt + ritual để mở khoá
- **Crafting prestige (L10+)** — sau max level, prestige rank → bonus quality roll
- **CraftSkill transfer** — mentor dạy trò tăng XP nhanh hơn
- **Item durability** — vũ khí mòn theo thời gian, cần sửa chữa tại forge
- **Set bonus** — trang bị đủ bộ (2/4 piece) → bonus thêm
