---
name: project-combat-redesign
description: Thiết kế lại combat gameplay (grid footprint + RTwP-ATB + AOE linear) - doc + core code
metadata: 
  node_type: memory
  type: project
  originSessionId: a3fd1a36-2990-4bcf-a01a-d5e73f230808
---

Combat đang được **thiết kế lại từ đầu** (2026-06-16), thay phần "combat test" cũ. Doc canonical:
`Assets/Scripts/Nizhangjie/M_COMBAT_DESIGN.md`.

**Quyết định chốt với user:**
- **Time model = GIỮ RTwP có sẵn** (`Combat/RTwP/CombatRTwPController` + `ActionGaugeSystem`).
  KHÔNG tạo hệ time-model mới. Running = AutoAttackAI tự đánh theo SPD; PAUSE = gauge nạp + queue lệnh
  (`PauseCommandQueue`) → Resume execute. (User chốt: "chỉ để lại cái RTwP thôi" — tôi từng đẻ dư
  `AtbController` rồi ĐÃ XÓA.) Core mới cắm LÊN TRÊN RTwP, không thay.
- **Bỏ hệ phụ** ở bản core: QiShield / LingLi / KillCombo / Knockback (thêm lại sau nếu cần).
- **GridManager "singleton" trong spec → KHÔNG làm** (vi phạm luật No-singletons). Dùng `CombatGrid`
  (plain C#, do CombatEncounter sở hữu) — đã mở rộng hỗ trợ footprint nhiều ô + rotation.
- **Damage**: công thức linear-diminishing AOE (`Σ coeff(i)`, coeff=max(floor,1-(i-1)*step),
  default step/floor=0.2) chỉ là **multiplier theo số ô trùm**, NỐI TẦNG vào DamageCalculator có sẵn
  (DEF/Resist/Crit/ArmorPen/Lifesteal). Crit/Evasion roll 1 lần/entity, AOE đất không bị Evasion né.
- **Determinism**: thêm `Combat/Core/DeterministicRng.cs`; DamageCalculator nhận param rng (default
  null = fallback UnityEngine.Random cho code cũ). Combat thật phải truyền rng.

**Đã code (milestone C1) — plain-C# testable:**
`DeterministicRng`, `Combat/Grid/GridFootprint.cs` (+ FromOffsets cho boss hình bất kỳ), `CombatGrid`
footprint (PlaceFootprint/RemoveFootprint/GetFootprint, PlaceUnit/RemoveUnit giờ delegate),
`Skills/HitShapes/OffsetListShape.cs`, `Damage/AoeDamageResolver.cs`, `CombatUnitDef` thêm
Size/FootprintOffsets/ComputeFootprint. Test: menu **Nizhangjie/Run M Combat Core Tests**
(`Tests/Editor/MCombatCoreTest.cs`). CHƯA chạy trong Unity Editor — cần verify.
(KHÔNG có AtbController — đã xóa, dùng RTwP.)

**Đưa lên Unity (vertical slice):** `Combat/CombatCoreDriver.cs` — launcher GỌN (không prefab/Canvas):
tái dùng CombatEncounter+GridVisual+UnitVisual, units dựng bằng code, HUD OnGUI, **wire vào
`CombatRTwPController` có sẵn** (Initialize sau Begin, SPACE pause/resume, 2 team auto-fight). Bản full
input/HUD là `CombatSceneBootstrap` — đừng gắn cả 2 cùng scene.
Guide: `Nizhangjie/M_COMBAT_UNITY_GUIDE.md` (scene/grid Tilemap/driver/HUD/controls).
LƯU Ý: namespace `Nizhangjie.Combat.Input` che `UnityEngine.Input` → phải fully-qualify `UnityEngine.Input`.
ServiceLocator.TryGet<T>() trả về T (null nếu chưa có), KHÔNG phải out-pattern.

**Unit/prefab data-driven (chỉnh chu, 2026-06-16):**
- `Data/Definitions/CombatUnitDefinition.cs` (POCO IDefinition) = unit COMBAT cụ thể (HP/ATK/size/
  footprint/visual). KHÁC `UnitDefinition` (template sinh đệ tử theo stat-range — đừng nhầm).
- Roster JSON `combat_units/` ở CẢ `Assets/Data` (Editor đọc) + `StreamingAssets/Data` (build): 6 mẫu
  (player_placeholder, beast_wolf, beast_boar, spirit_archer, construct_golem 2x2, demon_brute).
- `Combat/Units/UnitFactory.cs`: EnsureLoaded/Get/CreateUnit/BuildStats/FootprintOffsets/EquipSkills/
  SpawnVisual. Parse faction string→Faction.
- `Combat/Visual/UnitHealthBar.cs`: thanh máu sprite neo-trái, KHÔNG cần Canvas. **Dựng bằng code** ở
  UnitFactory.BuildHealthBar (child "HealthBar", tham chiếu trực tiếp qua Setup — KHÔNG transform.Find,
  KHÔNG nằm trong prefab) vì bản prefab cũ bar hay ẩn. Chết → ẩn riêng bar (không đụng unit root).
- `Combat/Editor/CombatPrefabBuilder.cs`: menu **Nizhangjie/Combat/Build Placeholder Assets (Prefab + Tiles)**
  → sinh `placeholder_square.png` + `Unit_Placeholder.prefab` + `Tiles/Tile_Floor|Wall|Highlight.asset`
  (cho GridVisual). Guide chi tiết click-by-click: `Nizhangjie/M_COMBAT_UNITY_GUIDE.md`.
- CombatCoreDriver giờ spawn từ roster qua UnitFactory + prefab (bỏ dựng GO bằng tay).
- KHÔNG asmdef trong Scripts → tất cả Assembly-CSharp, "Editor" folder tự editor-only.

**Footprint multi-ô (đã làm phần ĐẶT, 2026-06-16):** CombatEncounter.RegisterUnit có overload (unit,
anchor, cells) → Grid.PlaceFootprint. UnitFactory.FootprintSize/FootprintCells. GridVisual.FootprintCenterWorld
+ CellSize. UnitVisual.SetFootprint (canh giữa theo block). Prefab: Body tách thành CON riêng (root scale=1,
body scale theo size), bar nâng lên đỉnh + giãn bề rộng. golem visualScale=0.9 (factory tự ×size). →
unit 2×2 chiếm đúng 4 ô + khít block. visualScale giờ = tỉ lệ LẤP mỗi ô (0.8-1.0), KHÔNG phải scale tuyệt đối.

**Nhịp tick + movement (2026-06-16):** `CombatTickClock.TickRate` đổi mặc định 4→**10 tps**. Thêm
`StatType.MoveSpeed` (%, mặc định 100, append-only sau DesireRate). Công thức: ô/giây = MoveSpeed/50
(50%=1 ô/s, 100%=2 ô/s); hằng `TickMover.PercentPerCellPerSecond=50`. `Combat/Movement/TickMover.cs`
(tích progress mỗi tick → bước ô, hỗ trợ phân số) + `GridMovementSystem.cs` (sở hữu mover, tick theo
clock.OnTick, MoveTo(unit,cell) BFS). `UnitMovement.StepTo(cell)` bước 1 ô + fire OnMoved.
CombatUnitDefinition.moveSpeed; factory SetBase(MoveSpeed). MoveSpeed KHÁC SPD (SPD=nhịp đánh/gauge).
CHƯA wire vào AutoAttackAI (nó còn tự move kiểu cũ) — tích hợp là bước kế. Test MCombatCoreTest #8.

**Gauge giờ nạp theo TICK (sửa 2026-06-16):** ActionGauge nạp trong OnGameTick (running) bằng
`TickPause(units, TickInterval)`; BỎ nạp lúc pause (TickPaused giờ rỗng). Pause = CombatTickClock ngừng
→ gauge + di chuyển đóng băng. (User muốn: dừng game thì tick ngừng, gauge không hồi khi pause.)

**Pathfinding + điều khiển player (2026-06-16):**
- `Combat/Movement/GridPathfinder.cs` — A* 4 hướng, **binary min-heap** (tự viết, không SortedSet),
  heuristic Manhattan, lazy-deletion, tie-break thứ tự chèn → DETERMINISTIC. **Footprint-aware**
  (2×2 không lọt khe rộng 1). + GetReachable (Dijkstra giới hạn). GridMovementSystem.MoveTo giờ dùng nó.
- `GridVisual.HighlightTiles(tiles, Color)` overload (tầm đánh xám đậm).
- `GridVisual.ImportObstacles(grid)`: đọc tile vẽ tay trên Tilemap_Walls → grid.BlockTile (vật cản A*).
  Driver gọi trước spawn+DrawGrid. **Di chuyển KHÔNG cần vẽ tile** — CombatGrid là lưới logic, ô trống =
  đi được; DrawGrid chỉ để nhìn. (User tưởng cần vẽ tile mới đi được — không phải.) Nếu click không đi:
  check Grid GO ở Position(0,0,0)+CellSize 1 (WorldToGrid giả định grid ở gốc), đã chọn unit, không pause.
- CombatCoreDriver REWRITE: roster theo ARRAY `_playerUnits`/`_enemyUnits` (RosterEntry{unitId,cell}),
  2 phe tuỳ ý. Điều khiển Team A: click chọn → highlight tầm đánh (độc lập tầm đi) → click ô đi (A* +
  TickMover, KHÔNG giới hạn khoảng cách). SetPlayerTeam(A) → player không chạy AI. Enemy = AutoAttackAI.
- Test MCombatCoreTest #9 (A*). attackRange lấy từ CombatUnitDefinition.attackRange.

**Quái di chuyển theo tick (2026-06-16):** MoveSpeed VỐN đã set cho quái nhưng AutoAttackAI cũ bỏ qua.
Đã: `CombatRTwPController.AutoAIEnabled` (cờ tắt AI cũ; gauge vẫn nạp). Driver tắt nó, register CẢ 2 phe
vào GridMovementSystem, chạy `OnEnemyTick` (sub clock.OnTick): quái tìm địch gần → `MoveToRange` áp sát
(re-path chỉ khi idle/target dời, tránh reset progress) → vào tầm thì đánh trực tiếp qua DamageCalculator
(KHÔNG dùng PerformBasicAttack vì IsActed khoá ở real-time) với cooldown = AttackIntervalFromSPD.
Pathfinder thêm `FindPathToRange` (A* tới trong bán kính goal, vì ô goal bị địch chiếm). GridMovementSystem.MoveToRange.

**Lệnh đánh player + tài nguyên + loot (2026-06-16):**
- Player attack: click chọn unit → click ĐỊCH = `_attackTarget[uid]`; OnEnemyTick đổi thành `OnCombatTick`
  (cả 2 phe): quái auto-nhắm, player theo _attackTarget → `RunCombat` (áp sát MoveToRange + đánh). Click ô
  trống = đi + huỷ lệnh đánh. Đánh trực tiếp DamageCalculator (không PerformBasicAttack vì IsActed).
- **Linh Lực (mana):** `StatType.MaxLingLi` (append) + `Combat/Units/UnitMana.cs` (pool spend/restore, owned
  bởi CombatUnit, max từ stat). CombatUnitDefinition.maxLingLi. Chưa tiêu (chờ skill).
- **Khí Thuẫn (shield):** thêm vào `UnitHealth` (MaxShield/CurrentShield/SetupShield); TakeDamage hấp thụ
  thuẫn TRƯỚC, tràn vào HP; chết chỉ khi HP<=0. Backward-compat: MaxShield mặc định 0 = không thuẫn.
  **SỬA 2026-06-23 (bug "chớp đỏ mà không mất máu"):** factory TRƯỚC ĐÂY cấp thuẫn=maxHP cho MỌI unit → đòn vào
  thuẫn, HP không đổi → user tưởng vô hiệu. Giờ **opt-in**: thêm `CombatUnitDefinition.maxShield` (mặc định 0);
  `UnitFactory.CreateUnit` chỉ SetupShield khi maxShield>0. Chỉ warchief_heavy có (150, Chân Khí Giáp). `UnitHealth.AddShield`
  (skill heal cấp thuẫn). → đa số unit đánh vào thẳng HP.
- **Loot:** `Data/Definitions/MaterialDefinition.cs` + 4 JSON `Data/materials` (beast_core/beast_hide/
  spirit_herb/spirit_stone_shard, sync 2 dir). CombatUnitDefinition.drops (DropEntry{materialId,chance,min,max}).
  Event `EnemyLootDroppedEvent`+`LootStack` (CombatGameplayEvents). Driver sub CombatUnitDiedEvent →
  roll drops (DeterministicRng) → publish + log. Quái đã gán drops.
- TickMover.OnTick guard unit chết (tránh re-place lên grid). Test #10 (shield).

**Skill "Kiếm binh pháp" (2026-06-16):** 4 SkillDefinition JSON (Data/skills, sync 2 dir):
kbp_chem_ngang (AOE 3 ô ngang trước mặt), kbp_dap_truoc (knockback), kbp_dam_manh (bleed DoT),
kbp_dap_kiem (stun). `Combat/Skills/SkillCaster.cs` (static): FrontCells (hộp trước mặt = range×width
theo facing snap-4) → AoeDamageResolver (chỉ địch) → on-hit knockback (StepTo lùi) + status
(onHitEffectTag bleed/stun). Tiêu Linh Lực (Mana.Spend; thiếu → fail). CombatUnitDefinition.skills
(List<string>, map phím 1-4); player_placeholder gán 4 skill này. Driver: phím 1-4 chọn skill →
click hướng cast; cooldown theo tick (_skillCdTick). **Status tick 1 lần/giây** (tick%10) gọi
StatusEffectController.OnTurnStart (DoT bleed + giảm CC). Stun gate: RunCombat + TickMover dừng khi
IsStunned/IsFrozen. LƯU Ý: bleed/DoT cũng đi qua Khí Thuẫn trước (mọi damage qua shield). Test #11.

**SCENE TEST "combat_in_prototype" — PHÁT TRIỂN THÀNH TEST HARNESS (2026-06-23, đang làm):** mục tiêu test
mọi thứ combat + behaviour quái rồi ném thuật toán vào local/dungeon. Lộ trình 5 phase user chốt (làm cả 3
ưu tiên, mình tự sắp; chuẩn hoá JSON): **P0** dọn content → **P1** tách `CombatRuntime` khỏi driver → **P2**
AI quái thành module goal-selector kiểu `BeastAI` (`combatBehaviour` data-driven trên CombatUnitDefinition:
aggressive/skirmisher/caster/territorial/pack/flee_low_hp) — cái lõi ném vào dungeon → **P3** harness UX
(scenario JSON + nút spawn/reset/step/slow-mo + đổi behaviour) → **P4** debug overlay (AI state/target line/
path/damage number thay OnGUI). **CHẨN ĐOÁN gốc "chưa rõ":** `CombatCoreDriver` ôm 3 vai (dàn cảnh+runtime+
AI inline `OnCombatTick`/`RunCombat`/`NearestEnemyOf`) → AI quái không phải module nên không tái dùng được.

**P0 ĐÃ XONG (2026-06-23):** phát hiện 2 hệ combat tách theo SCENE — cũ `CombatTest.unity` (CombatSceneManager/
Bootstrap/EncounterPresets/CombatUnitDef) vs mới `combat_in_prototype` (CombatCoreDriver/UnitFactory/JSON). Đã
**dời cả stack cũ ra `<repo>/_LegacyCombat/`** (ngoài Assets, KHÔNG xóa cứng vì repo không git; 21 file+meta,
khôi phục = move ngược) — chi tiết ở CLAUDE.md §10. Trước khi dời đã **cứu 6 archetype sang JSON** (cả Assets/Data
+ StreamingAssets): disciple_swordsman(Hỏa melee, dùng kbp_*)/disciple_archer(Băng kiter range3)/disciple_brawler
(Thổ tank)/bandit_light/warrior_medium/warchief_heavy(boss). Skill cone/cross/line KHÔNG port được (SkillCaster
mới chỉ FrontCells) → archer/brawler tạm `skills:[]` đánh thường, kit dựng lại ở P2/P3. Verify parse+refs OK.
CÒN vướng keeper (chưa gỡ, follow-up): AutoAttackAI/Shield/Energy/Knockback/Commands/CombatTerrain/SkillExecutor.

**P1 ĐÃ XONG (2026-06-23):** tách `Combat/Runtime/CombatRuntime.cs` (orchestrator plain C# gói encounter+
RTwP+movement+loot, dựng GO con cho 2 MonoBehaviour). `CombatCoreDriver` rewrite thành vỏ harness (roster
Inspector + input + OnGUI + camera). API: ctor→ImportObstacles→SpawnRosterLogic→Begin(playerTeam)→SpawnVisuals→
Dispose; expose Encounter/Grid/Rtwp/MoveSys/Behavior/Rng/Spawned, event OnEnded/OnLoot. RosterEntry chuyển vào
CombatRuntime (driver dùng `CombatRuntime.RosterEntry[]`). BẪY đã gặp+sửa: `CombatUnitDiedEvent` ở
`Nizhangjie.Core.Events.SampleEvents` (KHÔNG phải Combat.Events) — thiếu using → CS0246.

**P2 ĐÃ XONG (2026-06-23) — AI quái thành module goal-selector (mirror BeastAI):** `Combat/AI/`:
- `CombatAI.cs` — `CombatAIContext` (perception: NearestEnemy/HpFraction/Dist/StepAway/FleeTarget, refresh mỗi
  tick), enum `CombatGoalKind{Idle,Approach,Attack,Kite,Flee,CastSkill}`, `CombatDecision`, `CombatBehaviourProfile`
  (parse string→kind+senseRange/kiteMinRange/fleeBelowHpPct/prefersSkills), `ICombatGoal`, `CombatGoalSelector`
  (global stateless, BuildDefault).
- `CombatGoals.cs` — FleeLowHp(100)/CastSkill(80 DORMANT, hook P3)/Kite(70)/AttackInRange(60)/Approach(50)/Idle(0).
- `CombatBehaviorSystem.cs` — EXECUTOR: tự sub TickClock.OnTick; quái→selector, player→lệnh đánh (SetAttackCommand/
  ClearCommand) PlanForcedTarget áp sát+đánh; tick status DoT/CC 1/giây; move qua MoveSys (re-path guard), đánh qua
  DamageCalculator (cooldown SPD). Event OnDecision + TryGetDecision (hook overlay P4). Owned bởi CombatRuntime.Begin.
- `CombatUnitDefinition.combatBehaviour` (field append, default "aggressive"). Set: spirit_archer=skirmisher (range4),
  beast_rabbit=defensive. Roster mặc định driver: spirit_archer (kite) + beast_wolf (charge).
- Test: `Tests/Editor/MCombatAITest.cs` menu **Nizhangjie/Run M Combat AI Tests** (7 case plain-C#, KHÔNG cần Play:
  CombatUnit trần + CombatGrid). **CHƯA verify Unity compile/run — user check Console.**
- Driver giờ KHÔNG còn AI inline (OnCombatTick/RunCombat/NearestEnemyOf/TryAttack đã chuyển vào system).
- **KITE-COMMIT (2026-06-23, user yêu cầu, ĐÃ SỬA v2):** pure-kite khiến 1 melee không bắt kịp + lùi từng ô vô tận
  (player giữ pace thì dist không bao giờ vượt kiteMin → không commit). Fix v2: **lùi NGUYÊN KHỐI** `kiteRetreatDistance`
  ô (3-5, ~tầm bắn) MỘT PHÁT (FleeTarget), tới đích mới ĐỨNG `kiteCommitAttacks` đòn, đòn đầu windup
  `kiteFirstAttackDelayTicks`=**10 tick** rồi bắn. Blackboard `CombatAIMemory` per-unit (kiteState 0 idle/1 đang-lùi/
  2 commit, commitAttacksLeft, windupTicksLeft) sở hữu bởi system, goal đọc/ghi qua `CombatAIContext.MemoryOf`.
  KiteGoal gate "tới đích chưa" bằng `ctx.Moving` (system gắn = MoveSys.IsMoving). `MoveToDest(null)` = no-op (giữ lùi,
  không cắt). TryAttack trả bool (chặn lúc windup), Execute trừ commitAttacksLeft mỗi đòn trúng. `Refresh` 4 args
  (thêm memory); `CombatAIContext.IsMoving` Func set 1 lần ở system ctor. Test #8 (windup=10).
- **Q1 user — tách behaviour⟂data cho TAME:** `combatBehaviour` = AI khi THÙ ĐỊCH. CombatBehaviorSystem gate THEO TEAM
  (unit==playerTeam → chỉ theo lệnh, KHÔNG chạy goal-selector). Tame = đổi Team sang player → behaviour hoang dã tự
  tắt tick sau, không lẫn (pet-AI riêng làm sau). Đã ghi rõ ở doc CombatUnitDefinition.combatBehaviour + comment system.
- CÒN LẠI: **P4** debug overlay (target line/path/AI state/damage number — dùng OnDecision/TryGetDecision đã cắm sẵn).
  CastSkill goal còn dormant (bật khi quái có kit skill + SkillCaster hỗ trợ ngoài FrontCells).

**P3 ĐÃ XONG (2026-06-23) — harness UX data-driven (chưa verify Unity):**
- `Data/Definitions/CombatScenarioDefinition.cs` (IDefinition): id/displayName/gridWidth/Height + playerUnits/enemyUnits
  (SpawnEntry{unitId,x,y}) + blockedTiles (Cell{x,y}). KỊCH BẢN chỉ là bố trí; AI vẫn từ combatBehaviour của unit.
- 3 JSON `Data/combat_scenarios/` (+ StreamingAssets): scn_kite_vs_charge (default), scn_boss_ambush (tái dựng
  EncounterPresets: 3 đệ tử vs 2 phỉ+2 chiến binh+đầu lĩnh + 4 cột đá cover), scn_pack_hunt (thỏ defensive + sói/heo).
- `CombatTickClock`: thêm `TimeScale` (slow-mo: scale accumulator, GIỮ TickInterval → chậm đồng đều) + `StepOnce()`
  (ép 1 tick khi pause; gauge RTwP vẫn đứng vì OnGameTick gate — chấp nhận cho debug).
- `CombatRuntime.BlockTiles(cells)`; `CombatBehaviorSystem.SetBehaviour(unitId, behaviour)` (override lúc chạy).
- `CombatCoreDriver` rewrite harness: nạp scenario qua DataRegistry (GetAll, sort id) → BuildCurrent/BuildEncounter
  (teardown Dispose + rebuild); OnGUI controls Prev/Next scenario · Reset(R) · Pause(SPACE) · Step(F) · Slow-mo toggle ·
  override behaviour quái đang "soi" (click địch khi chưa chọn ally = _inspected). Không có scenario → fallback roster
  Inspector. Thêm `[RuntimeInitializeOnLoadMethod]` clear scenario registry mỗi Play (bẫy static sống dai).
- CHƯA test riêng (P3 = UI/data glue; data validate bằng PowerShell parse+ref+bounds OK).

**HỆ SKILL HỢP NHẤT (2026-06-23, chưa verify Unity):** user yêu cầu skill gồm định hướng/aoe/effect. KHẢO: đã
có primitive (IHitShape Line/Cone/Rect/Cross/OffsetList + AoeDamageResolver + StatusEffectController DoT/CC +
ElementalReaction) nhưng PHÂN MẢNH — SkillCaster mới chỉ làm 1 shape "front", bỏ qua hitShapeType. Build
`Combat/Skills/SkillSystem.cs` (static, hợp nhất, thay SkillCaster cho gameplay; SkillCaster GIỮ cho MCombatCore #11):
- **4 targeting mode** (field mới `SkillDefinition.targetingMode`): directional (front/line/cone từ caster) ·
  target_point (rectangle/cross/circle quanh ô click) · self (quanh caster) · single (1 ô click).
- `ResolveCells(skill, casterPos, aimCell, grid)` dispatch mode×hitShapeType → ô (dùng IHitShape sẵn có + FrontCells +
  CircleCells=GetTilesInRange Manhattan). `Cast(...)` → nếu damage: AoeDamageResolver (chỉ địch) + knockback + onHit
  status; nếu `affectsAllies`: heal/shield/buff cho ĐỒNG MINH (dedup entity).
- Effect đợt này (user chốt): damage/DoT/CC/knockback/reaction (sẵn) + **heal/shield** mới (`healFlat`/`healPercentMaxHp`/
  `shieldFlat` + `affectsAllies`; `UnitHealth.AddShield`). BỎ buff/debuff stat + dispel (để sau).
- Driver repoint SkillSystem (preview=ResolveCells, cast=Cast aimCell). kbp_* (hitShapeType "front", không targetingMode→
  default directional) GIỮ NGUYÊN hành vi.
- Content demo (cả 2 dir): 5 skill `skl_kiem_khi`(line)/`skl_lac_thach`(target_point rect 3×3 Thổ)/`skl_kiem_vu`(self
  nova)/`skl_nhat_kiem`(single nuke)/`skl_hoi_nguyen`(self heal+shield) + 2 unit `disciple_adept`(4 mode)/`disciple_medic`
  (heal) + scenario `scn_skill_test` (adept+medic vs cụm 4 boar). Test `Tests/Editor/MCombatSkillTest.cs` menu
  **Run M Combat Skill Tests** (5 case). CHƯA: AI cast (CastSkillGoal vẫn dormant — nối SkillSystem là bước sau);
  buff/debuff stat; single chưa áp Evasion/HitAccuracy.

**UI/UX combat (2026-06-23, đang làm — user muốn dùng scene combat_in_prototype làm testbed prefab UI):**
- **Quyết định cách làm:** UI MÀN HÌNH (panel/skillbar/topbar/damage text) = approach C "mình viết script controller +
  guide, USER dựng prefab/Canvas + wire" (mình KHÔNG sửa .unity/.prefab). UI WORLD-SPACE per-unit (health bar, icon
  countdown) = dựng bằng CODE như UnitHealthBar. UI tách rời combat, nối qua event/CombatRuntime (luật EventBus).
- **ĐÃ LÀM:** (1) **giảm tốc đánh còn 1 nửa** — `CombatTickClock.AttackSpeedDivisor` const 300→**600** (interval gấp đôi).
  (2) **Icon countdown đòn đánh** `Combat/Visual/UnitAttackIndicator.cs` (vuông xám + thanh sáng dâng từ đáy kiểu LoL,
  đầy=sắp đánh; sprite kiếm tùy chọn Resources "Combat/sword_icon") — dựng code, gắn trong `CombatRuntime.SpawnVisuals`
  cho MỌI unit, đọc % nạp qua `CombatBehaviorSystem.AttackCharge(unit)` = (tick−lastAttackTick)/interval. Giúp đọc nhịp
  đánh quái + thả diều.
- **4 WIDGET MÀN HÌNH ĐÃ VIẾT (2026-06-23, approach C — user dựng prefab):** `UI/Combat/` namespace `Nizhangjie.UI.Combat`:
  `SelectedUnitPanelUI` (HP/LL/thuẫn/status, Image Filled) · `SkillBarUI` (4 slot, cooldown overlay, OnSkillButton(int),
  icon Resources/Skills/<id>) · `FloatingCombatTextUI` (+nested FloatingTextItem; nghe DamageDealtEvent, clone Text bay
  lên; chỉ DAMAGE, heal chưa có event) · `CombatTopBarUI` (scenario/pause/step/slow/result). Bind PULL: tự Find driver,
  nghe `SelectionChanged`/`RuntimeBuilt`. **Nút wire OnClick Ở INSPECTOR** (controller chỉ expose hàm public — đúng
  feedback ui_button_wiring), KHÔNG AddListener.
- **Driver mở UI-facing surface:** `Runtime/Selected/GridVisualRef/IsPausedUI/SlowMo/ScenarioName/Count/Index`, event
  `RuntimeBuilt`/`SelectionChanged` (route qua `SetSelected`), lệnh `UiReset/UiNext/UiPrev/UiTogglePause/UiStep/
  UiSetSlowMo/UiSelectSkill`, `SkillCooldownPct(unitId,sk)`. OnGUI cũ vẫn còn (xoá khi UI mới ổn).
- Guide cầm tay: `Nizhangjie/M_COMBAT_UI_GUIDE.md` (hierarchy + bảng field←object + wiring OnClick + Image Filled + TMP swap).
- CHỮ = **TextMeshPro** (`TMP_Text`, đã đổi 2026-06-23 theo user "cho đồng nhất"); Image/Button/Toggle = UGUI. User
  phải Import TMP Essentials lần đầu. Guide M_COMBAT_UI_GUIDE.md đã update TMP (tạo UI ▸ Text - TextMeshPro). CHƯA:
  heal/shield floating text (cần HealEvent); skill tooltip; AI cast skill.

**CÂN BẰNG đợt 1 (2026-06-23, user "combat quá nhanh"):**
- **Giảm sát thương trên DATA** (script PowerShell sửa atk mọi combat_units, 2 dir, UTF-8 no-BOM): faction "player" → atk×0.4,
  còn lại → atk×0.3 (vd wolf 70→21, player_placeholder 85→34). Skill damage cũng giảm theo (mult × atk).
- **Né + Chặn** `Combat/Damage/CombatMitigation.cs`: Evasion 20% (né hẳn, 0 dmg) → nếu KHÔNG né mới roll Block 30%
  (còn nhận 30%) → LOẠI TRỪ lẫn nhau. FLAT mọi unit (tạm; sau đọc StatType.Evasion + stat block). Áp trong
  `CombatBehaviorSystem.TryAttack` cho **đòn thường** (skill/AOE chưa áp — AOE không bị né theo design). Lifesteal ×mitMult.
  Test MCombatSkillTest #6.
- **Không nạp pool đánh khi tiếp cận** (user yêu cầu): `CombatBehaviorSystem` reset `_lastAttackTick=tick` mỗi tick khi
  decision != Attack (approach/kite/flee/idle/CC + player no-command) → AttackCharge=0 khi đang đi; chỉ nạp khi ĐỨNG
  trong tầm (Attack) → đòn đầu phải sạc đủ, icon countdown bắt đầu từ 0 lúc áp sát. (User sẽ chỉnh số nhiều về sau.)

**ROADMAP combat cập nhật (2026-06-23):** doc §8 đối chiếu lại — C1-C5 gần như XONG (qua đường khác): C2 move (TickMover/A*),
C3 skill SkillSystem+UI Canvas (cast time/pause-queue CHƯA), C4 AI goal-selector (AI cast skill CHƯA), C5 balance đợt1.
Thêm milestone "depth": C6 Linh Lực regen + AI cast skill · **C7 phản ứng nguyên tố (XONG)** · C8 cast time/telegraph ·
C9 status depth (buff/debuff stat vào CharacterStats + icon + popup né/chặn/heal) · C10 boss nhiều ô (rotate/trượt).

**C7 XONG (2026-06-23):** `ElementalReaction` rewrite — `ApplyElementalHit(target,element,attackerId,bus)` = TryReact +
gieo AURA. Aura = marker nhẹ: fire→burn(DoT nhẹ), ice→chill, water→wet, lightning→electrify, wood→wood_mark, earth→
earth_mark (KHÔNG dùng freeze làm aura → tránh perma-CC). Payoff phản ứng: Fire+Ice=Tan Chảy(+10%HP), Ice+Lightning=Vỡ
Băng(stun), Ice+Water=Đóng Băng(freeze 2), Fire+Wood=Bùng Cháy(burn mạnh), Earth+Lightning=Nhiễm Từ(def_down), Fire+
Poison=Bộc Cháy. ICD `reaction_icd` 1 nhịp. Wire: TryAttack (đòn thường, bus null) + AoeDamageResolver (skill). Reaction
gây damage → DamageDealtEvent → floating text tự hiện. def_down chưa áp stat (chờ C9). Test MCombatSkillTest #7.

**Còn lại (core cũ):** Linh Lực regen; footprint "trượt" khi di chuyển; rotate; cast time; HUD Canvas; inventory
nhận loot; ICD reaction; bleed bypass shield? (cân nhắc); cân bằng. Cụm Shield/Energy/Knockback/Commands/AutoAttackAI
vẫn vướng CombatUnit/RTwP (gỡ = follow-up, xem CLAUDE.md §10).

**REFACTOR "global use" (2026-07-06, user yêu cầu: hoàn thiện prototype để đưa vào Wildmap — ưu tiên
"dựng hệ thống tương tác cho người chơi + UX để tạo prefab dùng chung + cải thiện flow nhịp độ"):**
- Tách tương tác người chơi khỏi `CombatCoreDriver` (trước ôm cả input LẪN scenario/roster/camera harness)
  ra `Combat/Player/CombatPlayerController.cs` (MonoBehaviour ôm chọn unit/tầm đánh/skill preview-cast/
  pause-step-slowmo/auto-pause) — gọi `Bind(runtime, gridVisual, playerTeam)` sau `CombatRuntime.Begin()`,
  KHÔNG biết gì về scene/scenario gọi nó.
- `Combat/Player/ICombatSession.cs` (interface) + `CombatPlayerController` tự đăng ký vào
  `ServiceLocator<ICombatSession>` mỗi lần Bind (Unregister rồi Register). 3 widget UI (SkillBarUI/
  SelectedUnitPanelUI/FloatingCombatTextUI) đổi từ `FindFirstObjectByType<CombatCoreDriver>()` sang
  `ServiceLocator.TryGet<ICombatSession>()` + `EnsureSession()` poll mỗi frame (phòng OnEnable UI chạy
  trước lần Bind đầu — thứ tự script không đảm bảo). → 4 widget này giờ là prefab "global use" thật: gắn
  vào bất kỳ scene nào (kể cả Wildmap) không cần `CombatCoreDriver`. `CombatTopBarUI` CỐ Ý GIỮ coupling
  với driver (Prev/Next/Reset scenario là tính năng harness-only, vô nghĩa ở Wildmap).
- `CombatCoreDriver` giờ CHỈ còn: scenario/roster fallback/camera/OnGUI debug + soi AI quái debug
  (`_inspected`, qua event `CombatPlayerController.EnemyClicked`). Awake tự `GetComponent`/
  `AddComponent<CombatPlayerController>`. Cách dùng combat KHÔNG cần driver: xem M_COMBAT_UI_GUIDE.md §9
  (code mẫu new CombatRuntime → Begin → AddComponent<CombatPlayerController>().Bind(...)).
- **Auto-pause-on-ready** (cải thiện nhịp độ — user chốt hướng "flow nhịp độ"): `CombatRTwPController`
  có field `_autoPauseOnReady` TỪ TRƯỚC nhưng CHẾT (chưa nối) — giờ nối thật: `CheckAutoPauseOnReady()`
  trong `OnGameTick` (sau `_actionGauge.TickPause`), cạnh lên false→true gauge unit PHE NGƯỜI CHƠI →
  `Pause()` 1 lần (HashSet `_wasReady` chống spam khi gauge đứng yên ở đầy). Public property
  `AutoPauseOnReady` get/set. Toggle qua `CombatPlayerController.UiSetAutoPauseOnReady` /
  `CombatCoreDriver.UiSetAutoPauseOnReady` / `CombatTopBarUI.OnAutoPauseToggle` (chưa wire Toggle trong
  Canvas — user tự thêm nếu muốn, giống SlowToggle). Mặc định TẮT.
- **CHƯA LÀM** (còn lại của "đưa vào Wildmap"): chưa có launcher/entry-point THẬT nối WildLocal scene →
  combat (trigger khi gặp yêu thú, dùng CombatRuntime+CombatPlayerController y hệt, quay lại wildmap sau
  khi thắng/thua/loot) — xem [[project-localmap-wildgen]] mục "Yêu thú SỐNG" (roadmap #3, chưa code).
  CHƯA verify compile trong Unity (Editor đang mở lúc code — user tự check Console).

**PawnInspector + Combat Hotbar + draft toggle (2026-07-06, cùng session — user muốn "đào lại PawnInspector
áp dụng vào prototype trước khi thêm hotbar"):**
- **Roster Disciple thật (shortcut)**: `CombatCoreDriver._useDiscipleTestRoster` (mặc định BẬT) — phe đồng
  minh spawn qua `DiscipleDatabase.CreateInstance` (4 JSON test cam/vàng/đen/lam có sẵn) thay vì
  CombatUnitDefinition thuần. `CombatRuntime.SpawnDiscipleUnit(disciple, combatDefId, team, cell)` +
  `GetDisciple(unitId)` hiện thực hoá convention `"disciple:{id}"` (đã có comment sẵn trong `CombatUnit.cs`
  từ trước nhưng chưa ai nối). Stats/visual VẪN lấy từ 1 CombatUnitDefinition cố định (`_discipleCombatDefId`,
  mặc định `disciple_swordsman`) — **CHƯA có bridge Disciple→CombatUnitDefinition thật** (map Aptitude/Force/…
  sang ATK/DEF/HP combat) — **TODO phải viết lại** khi rời prototype, cùng với pipeline spawn-từ-tông-môn
  thật (hiện chỉ test id cứng trong Inspector).
- **PawnInspectorController tái dùng**: `UI/Sim/Pawn/PawnInspectorCombatBridge.cs` (mới) — nghe
  `ICombatSession.SelectionChanged` qua ServiceLocator, tự `Bind(session.SelectedDisciple)` (null → tự ẩn,
  vd chọn quái không phải Disciple). Lý do cần bridge riêng: `PawnInspectorController.Bind` nhận `Disciple`
  (không phải UnityEvent) nên không wire OnClick được — đây LÀ code-level "Bind PULL" giống 3 widget combat
  khác. Việc CÒN LẠI (Editor, user tự làm): kéo cụm PawnInspector (PawnBar/Statusheet/SkillPage/InventoryPage/
  SheetTabBar) từ WildLocal/prefab sang scene `combat_in_prototype`, add bridge, tắt `_autoBindPlayer`.
- **"Combat toggle" (draft, kiểu RimWorld) — CHỐT với user:** OFF (mặc định) = thụ động, KHÔNG chủ động tấn
  công, CHỈ đánh trả khi bị đánh trước; ON = thủ công hoàn toàn (tắt auto-retaliate), MỞ hotbar. Code:
  `CombatBehaviorSystem` thêm `_drafted` dict + `SetDrafted`/`IsDrafted` + `NotifyDamaged(victimId,attackerId)`
  (auto `SetAttackCommand` khi unit phe người chơi OFF bị đánh và chưa có lệnh nào — dùng lại pipeline lệnh-
  đánh có sẵn, KHÔNG viết AI mới). `CombatRuntime.Begin` subscribe `DamageDealtEvent` → gọi `NotifyDamaged`.
  `CombatPlayerController.UiToggleDraft()` (0 tham số, wire OnClick được) + event `DraftChanged`. Nút dư trên
  PawnBar (ảnh user vẽ) → wire thẳng vào `CombatCoreDriver.UiToggleDraft`.
- **Hotbar mới `UI/Combat/CombatSkillHotbarUI.cs` — THAY hẳn SkillBarUI cho unit Disciple** (đã chốt với
  user): hiện khi `SelectedDisciple != null && IsSelectedDrafted`. Nội dung = `Disciple.Books` (mirror cách
  `PawnSkillPanel.BuildGroups` nhóm — DUPLICATE nhẹ logic đó, không sửa PawnSkillPanel) + nhóm "Bản Năng"
  (skill lẻ) luôn ở cuối vòng; 2 nút lên/xuống `OnBookNext`/`OnBookPrev` đổi bộ đang xem.
  **Slot skill KHÔNG phải mảng cố định** (user bác bỏ bản đầu dùng `Slot[6]` ẩn/hiện — yêu cầu "bao nhiêu
  hiện bấy nhiêu"): `UI/Combat/CombatSkillSlotUI.cs` (component prefab riêng, mirror `SkillRowUI`/
  `SkillBookRowUI` ở UI/Sim/Skills) được **Instantiate động** vào `_slotContainer` (Horizontal Layout Group,
  Control Child Size + Force Expand đều TẮT) đúng số skill của bộ đang xem, **giữ thứ tự gốc**
  `SkillBookDefinition.skillIds`. Click wire **bằng code** lúc `Bind()` (`Button.onClick.AddListener`)
  — KHÔNG Inspector OnClick (số lượng chỉ biết lúc chạy, đúng quy ước list sinh động sẵn có của dự án) →
  `ICombatSession.UiSelectSkillDefinition(SkillDefinition)` (method MỚI, chọn thẳng def thay vì index vào
  `CombatUnitDefinition.skills` như `UiSelectSkill(int)` cũ — 2 API song song, quái/unit không-Disciple vẫn
  dùng index cũ). **CHƯA LÀM**: phím tắt 1-4 vẫn trỏ `CombatUnitDefinition.skills` cũ (KHÔNG đồng bộ hotbar)
  — dùng chuột click slot là đường chính thức, nối phím tắt là follow-up.
- Guide chi tiết từng bước Editor: `M_COMBAT_UI_GUIDE.md` mục 10.
- **Fix bug click UI lọt xuống grid**: `CombatPlayerController.HandleSelectionInput` thêm guard
  `IsPointerOverUI()` (`EventSystem.current.IsPointerOverGameObject()`) đầu hàm — click trúng panel/hotbar/
  nút không còn bị hiểu thành lệnh di chuyển/tấn công trên grid nữa.
- **Slot skill Instantiate ĐỘNG** (user bác bỏ mảng cố định): `CombatSkillSlotUI.cs` (mới, mirror
  SkillRowUI) + `CombatSkillHotbarUI` sinh đúng số slot theo skill thật của bộ đang xem (giữ thứ tự gốc
  `skillIds` JSON), Horizontal Layout Group tự co giãn. Slot có thêm **`_accentIcon`** (Image riêng, KHÁC
  `_icon`) đổi màu theo `SkillDefinition.elementAffinity` qua bảng tĩnh `CombatSkillSlotUI.AccentColors`
  (chốt với user: Hỏa=cam, Thủy=xanh dương, Thổ=nâu đất, Kim=nâu vàng, Mộc=xanh lá, vật lý(None)=vàng;
  Phong/Lôi/Băng chưa chốt màu — tự bổ sung hợp lý).
- **Content mới — bộ "All Elemental" test**: `Data/skill_books/book_all_elemental.json` (+ StreamingAssets)
  gồm 8 skill mới `Data/skills/elem_{wind,water,fire,earth,lightning,ice,metal,wood}_strike.json` — mỗi
  cái CHỈ gây sát thương thường (damageMultiplier 1.3, single-target, không onHit effect/knockback/DoT),
  1 skill/hệ, elementAffinity khớp enum (Wind=1..Wood=8). Gán vào `disciple_test_den` (đệ tử đen, cả 2 dir)
  thêm entry `books: [..., {"bookId":"book_all_elemental","mastery":0}]` — giờ đệ tử đen có 2 bộ
  (`book_co_quyen` mastery 2 + `book_all_elemental` mastery 0), test được hotbar cycle 3 nhóm (2 bộ + Bản Năng).
- `CombatCoreDriver` fallback roster: `_playerUnits` giờ 3 slot (thêm cell (2,7)), `_discipleTestIds` giờ
  3 id (`cam`/`vang`/`den`) — để đệ tử đen (mang bộ All Elemental) spawn được mặc định khi KHÔNG có scenario
  JSON nào active. Lưu ý: nếu có scenario `Data/combat_scenarios/*.json` load sẵn, scenario đó override
  roster fallback này — phải chỉnh scenario hoặc tắt để test đúng disciple_test_den.

**Cast time thật + Tactical Slowdown thay pause+queue (2026-07-06, cùng session) — user chốt: bỏ hướng
"pause+command-queue" của design doc gốc §1, làm giống FF7 Remake hơn (chậm thời gian thay vì dừng cứng):**
- Bỏ hẳn phím tắt 1-4 chọn skill (xung đột hotbar Disciple mới) — chọn skill giờ CHỈ qua click chuột
  (hotbar hoặc SkillBarUI cũ). `UiSelectSkill(int)` method vẫn còn, chỉ bỏ binding bàn phím.
- **SPACE đổi nghĩa**: từ hard-pause (`TogglePause`) sang **giữ = chậm 1/10** (`CombatPlayerController.Update`
  set `TickClock.TimeScale=0.1` lúc GetKeyDown, trả về 1f/0.25f lúc GetKeyUp) — không dừng cứng, chỉ chậm hẳn
  để đọc tình huống, đúng tinh thần FF7R Tactical Mode. Hard-pause vẫn còn (nút Pause TopBar/OnGUI, debug),
  chỉ không gán SPACE nữa.
- **`Combat/Skills/SkillCastController.cs` (mới)** — cast time THẬT thay vì tức thời: `BeginCast` gieo skill,
  unit bị khoá (`CombatBehaviorSystem` gate y hệt stun qua param mới `Func<string,bool> isCasting` ở
  constructor) chờ N tick rồi `SkillSystem.Cast` mới chạy thật (mana trừ lúc NÀY, không phải lúc BeginCast).
  N = `castTime`(giây, field JSON có sẵn từ trước)×TickRate, **sàn tối thiểu 2 tick** nếu skill chưa khai
  castTime (đa số skill test = 0 → dùng sàn) — skill khai castTime riêng cao hơn (vd Liệt Hỏa Trảm 0.5s=5
  tick) giữ nguyên giá trị riêng, KHÔNG bị đè. Owned bởi `CombatRuntime.Casting`, tick theo `CombatTickClock`.
- **`Combat/Visual/UnitCastBar.cs` (mới)** — thanh cast world-space trên đầu unit (mirror
  `UnitAttackIndicator`, dựng code trong `CombatRuntime.SpawnVisuals`, KHÔNG cần wire Editor), ẩn mặc định,
  hiện + fill trái→phải theo `Casting.GetProgress(unitId)` khi đang cast.
- **CHƯA LÀM (follow-up)**: cast chưa bị ngắt bởi stun/CC giữa chừng (chỉ chặn BẮT ĐẦU cast mới lúc đang
  stun); di chuyển bằng click lúc đang cast có thể nhích 1 tick trước khi bị khoá hẳn (MoveTo gọi trực tiếp,
  không qua gate mỗi-tick) — chấp nhận cho bản test.
- Guide chi tiết: `M_COMBAT_UI_GUIDE.md` mục 11.

Liên quan: [[project-nizhangjie]], [[project-nizhangjie-gotchas]], [[project-localmap-wildgen]], [[project-disciple-database]].
