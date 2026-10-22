---
name: project-data-registry
description: "Kiến trúc data-driven/modding — GameDatabase + Def + namespace:id + resolve pass + Newtonsoft. Canary BeastSpecies ĐÃ CODE + TEST PASS 13/13 trong Unity (2026-06-26)."
metadata: 
  node_type: memory
  type: project
  originSessionId: a17cd870-07ec-4d26-9ac2-c62d7a890063
---

**Kiến trúc data-driven nền tảng.** Doc canonical: `Assets/Scripts/Nizhangjie/M_DATA_REGISTRY_DESIGN.md`
(xem §9c cho đợt code mới nhất). Bối cảnh mới: user đưa "Nishangjie blueprint" (sandbox modding engine) —
phần lớn TRÙNG design này; đánh giá đầy đủ + 2 landmine (Dictionary cần Newtonsoft; JSON Patch hoãn) đã nói với user.

**ĐÃ CODE + TEST PASS 13/13 trong Unity 2026-06-26 (canary):**
- GOTCHA Unity gặp khi tích hợp: file .cs tạo từ ngoài lúc Unity đang refresh → "ghost script" (có .meta nhưng
  KHÔNG vào `Assembly-CSharp.rsp` → type "could not be found" dù code đúng, recompile vô ích). Cách chẩn đoán:
  đọc `Library/Bee/artifacts/<dag>/Assembly-CSharp.rsp` xem file có trong list không. Fix: xóa .meta + xóa
  `Library/ScriptAssemblies` & `Library/Bee` rồi mở lại Unity (full rebuild). Restore Newtonsoft xong mới compile được.
- **QUYẾT ĐỊNH MỚI của user (lật phần JsonUtility):** chuyển nền sang **Newtonsoft.Json** (`com.unity.nuget.newtonsoft-json@3.2.1`
  trong manifest). Opt-in per-type qua `[UseNewtonsoftJson]` (`Data/Loaders/UseNewtonsoftJsonAttribute.cs`) — type
  gắn → Newtonsoft (Dictionary/nested); không gắn → JsonUtility như cũ. **Newtonsoft KHÔNG tôn trọng `[NonSerialized]`
  → phải `[JsonIgnore]`** (đã thêm vào `Def.defName/modSource/Id`, `BeastSpecies.resolvedCombatUnit`).
- `GameDatabase.cs` (composition root, `: IDefResolver`): LoadAll (hiện chỉ `combat_units`+`beasts`, mở rộng dần),
  ResolveAndValidate(strict)→throw `GameDatabaseException`, Freeze, Get<T>. Resolver raw-fallback cho def IDefinition chưa stamp.
- `JsonDataLoader`: provenance `List<(path,ns)>`, stamp `defName="ns:id"` cho Def-based, `NormalizeNamespace`. `ModLoader` truyền ns=tên folder.
- Canary = `BeastSpeciesDefinition` (+`extraStats` Dictionary, json boar có `ferocity:1.5`). `GameBootstrap` gọi LoadAll/Resolve/Freeze (+bool `_strictDataValidation`).
- Test `Tests/Editor/MDataRegistryTest.cs` → menu `Nizhangjie/Run M Data Registry Tests`.
- LoadAll ĐÃ MỞ RỘNG 13 type (combat_units/skills/materials/status_effects/combat_scenarios/beasts/units/traits/equipment/recipes/factions/buildings/local_features). Test 18/18.
- ModLoader ĐÃ LÀM: `ModInfo.cs` (mod_info.json: dependencies cứng + loadAfter/loadBefore mềm, id 'core' reserved), `ModLoader.ResolveLoadOrder` PURE topo-sort Kahn (prune thiếu dep fixpoint + cycle detect, 1 mod hỏng không brick), template `StreamingAssets/Mods/_EXAMPLE_mod_info.json`, test `MModLoaderTest` menu `Run M ModLoader Tests` (11 ca).
- LocalizationManager ĐÃ LÀM: `Localization/LocalizationManager.cs` (static, string-ID→text, nguồn `Data/lang/{code}.json` merge base+mod, fallback vi, event LanguageChanged), `JsonDataLoader.GetExistingPaths`, `Data/lang/vi.json`+`en.json` (UI_STAT_ENERGY="Linh Lực"/"Spirit Energy"), wired GameBootstrap, test `MLocalizationTest`.
- **→ BLUEPRINT KHÉP:** agnostic registry+Newtonsoft ✓ namespace:id+resolve ✓ ModLoader topo-sort ✓ Localization ✓. Tùy chọn còn: mở rộng LoadAll, migrate def→Def, gỡ hardcode `==` chuỗi combat, JSON Patch (hoãn).

---

**Thiết kế gốc — ĐÃ CHỐT (2026-06-22):**

**Tầm nhìn user:** Nizhangjie = ENGINE, logic tách HOÀN TOÀN khỏi data (giống Minecraft/Terraria/RimWorld);
mọi unit/quái/skill/vật phẩm/vùng đất là content JSON, mod người dùng drive y như base game. Tổng hợp lúc boot.

**Hạ tầng đang có (đã đọc, đúng hướng):** `Data/Loaders/DataRegistry.cs` (static-per-type, Register override
theo Id), `JsonDataLoader.cs` (đọc Assets/Data ở Editor / StreamingAssets ở build, mod paths ưu tiên sau base),
`Modding/ModLoader.cs` (quét StreamingAssets/Mods/*/Data, sort tên), `Core/Bootstrap/GameBootstrap.cs`
(ServiceLocator.Clear + GlobalEventBus + GameClock), `Data/Definitions/IDefinition.cs` (interface chỉ có `Id`).

**6 quyết định CHỐT:**
1. **GameManager + GameDatabase** = composition root, mở rộng GameBootstrap, gom toàn bộ lúc boot.
2. **Option A typed** `Get<T>(id)` + **namespace:id** (`core:beast_wolf`). Namespace IMPLICIT theo folder mod
   (base=`core`, mod=tên folder lowercase). Ref không có `:` → default `{owner.modSource}` rồi `core`.
3. **namespace tách ADD/OVERRIDE** tự nhiên (mod tự namespace=thêm; ghi đúng `core:x`=sửa). Patch xpath ĐỂ SAU.
4. **`Def` abstract class thay `IDefinition`** — base có id/displayName/description (author) + `[NonSerialized]`
   defName/modSource (loader stamp) + hook `ResolveReferences(IDefResolver)` + `ConfigErrors()`. JsonUtility
   serialize field kế thừa OK (luôn deser vào type cụ thể). Migration mechanical ~15 file, làm SỚM.
5. **Resolve + validate pass lúc boot** (bước "amber") = thứ thay "compile check" cho data. Đổi string-ref →
   `[NonSerialized]` object-ref + validate; thiếu ref/sai type/sai enum → CHẶN boot + báo cáo rõ mod nào.
   Win phụ: enum string (statType/modifierType) hiện parse-lúc-apply fail âm thầm → chuyển validate sang boot.
6. **Codegen id constants** (`Defs.Skill.KbpChemNgang`) = tùy chọn, làm SAU nếu cần compile-time thật cho base.

**QUAN TRỌNG — đã giải thích cho user:** namespace:id KHÔNG cho compile-time safety (string không qua
compiler). Lợi ích thật = chống va chạm mod + load-time fail-fast + provenance. (User tưởng giúp "tìm lỗi compile".)

**Logic đang hardcode content cần bỏ dần:** `entityKind=="beast"`, `case "bleed"/"stun"`, `skillType=="active"`,
`faction=="hostile"` → đọc từ Def thay vì `==` chuỗi.

**Thứ tự build:** ①Def base+migrate 15 file ②loader stamp namespace ③IDefResolver+GameDatabase ④wire
ResolveReferences/ConfigErrors ⑤GameManager ⑥(sau) BeastSpeciesDef cắm vào + codegen + patch. Test mới
`Tests/Editor/MDataRegistryTest.cs`.

**ĐÃ TẠO đợt đầu (2026-06-22, CHƯA verify compile Unity):**
- `Data/Definitions/Def.cs` — base + `IDefResolver` (Resolve `where T:class,IDefinition`, coexist def cũ).
- `StatusEffectDefinition.cs` + 9 JSON `Data/status_effects/` (bleed/burn/poison/stun/freeze/regen/atk_up/
  def_down/wet). Map 1:1 hardcode trong `Combat/StatusEffects/StatusEffectController.cs` (TickEffect switch
  `case poison/burn/bleed`, IsDebuff list, IsStunned/IsFrozen) + ElementalReaction marks (wet...). Field:
  tickBehaviour/magnitudeKind/preventsAction/preventsMove/isDebuff/bypassShield/statModifiers. → bước kế: đọc
  def thay switch.
- `BeastSpeciesDefinition.cs` + 3 JSON `Data/beasts/` (species_lang_forest, species_boar_plains,
  event_bloodmoon_brute). **QUÁI SỰ KIỆN = `spawnContext` "wild"|"event"|"boss" + `eventTags[]`**: event →
  spawner bỏ qua pool, hệ Event gọi `Get(Id)` hoặc lọc eventTags. ResolveReferences resolve combatUnitId.
- JSON sync CẢ Assets/Data + StreamingAssets/Data (24 file, ConvertFrom-Json OK). "Vùng sát thương" KHÔNG cần
  def mới — đã ở SkillDefinition (hitShapeType/Range/Width); quái AOE = normalAttackId→SkillDefinition.

**CharacterStats review (2026-06-22):** math `(base+Σflat)×(1+ΣpAdd)×Π(1+pMul)` ĐÚNG, khớp doc. Rủi ro thật
KHÔNG ở công thức mà ở: ①KHÔNG có giá trị mặc định — CritDMG (doc=1.5)/MoveSpeed(100)/HitAccuracy nếu factory
quên SetBase → Get trả 0 ÂM THẦM. ĐÃ XÁC NHẬN footgun: DamageCalculator dòng 77 `Get(CritDMG) ?? 1.5f` chỉ
guard khi stats=null, KHÔNG guard khi CritDMG base chưa set (Get trả 0, không null) → crit ×0 = 0 dmg. Nên
data-drive bảng StatDefaults. ②resist: ĐÃ VERIFY — KHÔNG bug. DamageCalculator.GetResistance chia /100f
(0-100 scale → 0-1 đúng), Mathf.Max(1,...) min 1 dmg. Lo cũ sai, bỏ.

Liên quan: [[project-nizhangjie-overview]], [[project-monster-design]], [[project-combat-redesign]].
