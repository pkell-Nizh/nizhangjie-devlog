---
name: project_m66_tensura_pack
description: "M66 content-pack tensura: (Naming + Skill + stat khối) — spec + code STAGED, chưa apply"
metadata: 
  node_type: memory
  type: project
  originSessionId: 8beb6837-4902-46bb-93a3-9e302fea863a
  modified: 2026-09-06T22:20:14.452Z
---

Content-pack `tensura:` (cơ chế Naming của *Tensei Shitara Slime* + hệ skill 6 bậc + stat khối riêng),
ngang hàng `Sim/Xianxia/`. **KHÔNG tái dùng stat/tài nguyên xianxia** (magicule ≠ Linh Lực, ep ≠ Cảnh Giới).

## ⚠ 2026-09-28: CẤT THÀNH MOD TẮT (commit 5550222)
Data nay ở `StreamingAssets/Mods/tensura/` (namespace `tensura:`, `mod_info.enabled=false`), đã gỡ khỏi CẢ 2
thư mục Data. Code giữ nguyên + compile; `TensuraContentPack.Install`, LoadType 2 folder, save IO đều gate
`ModLoader.IsLoaded(TensuraContentPack.ModId)`. Capability id đổi `tensura:tensura_can_name`. Bật lại = sửa
`enabled:true`. ⚠ Khi bật lại cần soát: race id so chuỗi thô (`tensura_goblin`) + `RaceOps` fallback `core:`,
ThoughtRegistry khoá theo `d.id` thô. Test M66 (def dựng tay) vẫn xanh.

## Trạng thái (2026-09-06) — ĐÃ APPLY + VERIFY
Đã copy vào `Assets/Scripts/Tensura/` (15 cs) + `Assets/Scripts/SaveSystem/` (2 IO) + tests, JSON vào CẢ
`Assets/Data` LẪN `Assets/StreamingAssets/Data`. Patch: `GameDatabase.LoadAll` (+2 LoadType),
`SaveData` (+2 field), `SaveManager` (3 chỗ ×2 IO), `GameRoot` mục 6c-quater (`TensuraContentPack.Install`),
`CorePawnInteractions` (+verb "BestowName" / "Đặt Tên").
- **Compile: 0 error / 0 warning.**
- **Tests: M66 Body 20/20 · Naming 21/21 · Skill 16/16 · hồi quy M37 Decoupling 42/42 (Arch_CoreNeverDependsOnSim xanh) · Research all PASS.**
- **Play Mode 2026-09-06:** `[Tensura] Đã cắm content-pack` ở boot, 0 error; `IsInstalled=True`, 2 registry
  + RebirthTracker trong ServiceLocator; avatar "Đệ Tử Cam" tự nhận `core:tensura_can_name`; 6 evo + 12
  skillInfo nạp qua GameDatabase. End-to-end `TensuraNamingFlow.Begin`: ogre→**tensura_kijin**, +2 intrinsic
  skill, soul=Complete, bond ghi, vào party avatar, magicule 500→31.7 + strain 468.
- Bản staged nguồn: `_StagedTensura_M66/` (repo root, ngoài Assets) + `scratchpad/tensura_pack/`.

## Bổ sung 2026-09-06 (đợt 2) — sân thử + quái + panel
- **Panel `TensuraNamingPanel`** (IMGUI/OnGUI, tự dựng, KHÔNG prefab): nhập tên + chọn Equal/MasterSub +
  ý đồ + xem BÁO GIÁ trực tiếp (cost/rủi ro %/kiệt sức/block). `TensuraNamingFlow` tách `Begin` (mở panel)
  ⟂ `Execute(ctx,name,mode,intent)` (làm thật). Panel uGUI đẹp = polish sau.
- **`CombatUnitDefinition.raceId`** (field MỚI, rỗng="beast") + `BeastActor.EnsureBody` đọc nó → beast
  mang `PawnBody.Race` riêng. 2 file core sửa THÊM (additive).
- **3 quái test**: `combat_units/tensura_beast_{goblin,wolf,beetle}.json` + `beasts/tensura_species_{goblin,wolf,beetle}.json`
  (raceId `tensura_goblin`/`tensura_direwolf`/`tensura_beetle`, tameable, dangerBand 0-1).
- **Beetle chain MỚI**: race `tensura_beetle`→`tensura_armor_beetle` + `evo_beetle_armorbeetle` + skill
  `tensura_skill_hard_shell` + `info_hard_shell` (7 evo, 13 skill_info giờ).
- **Fixed map (M33)** `Data/fixed_maps/tensura_naming_testground.json` — "Sân Thử Đặt Tên" 28×13, kín,
  entry/exit, 4 goblin + 3 wolf + 3 beetle vô danh. `ConfigProblems=0`, `Run FixedMap Tests` 42/42.
- **Vào sân thử**: menu `Nizhangjie/Tensura/Enter Naming Testground` (`Editor/TensuraTestMenu.cs` +
  runtime `TensuraTestBootstrap.cs` dùng `SessionState`+`[RuntimeInitializeOnLoadMethod(BeforeSceneLoad)]`
  vì `FixedMapEntryContext` static bị domain-reload xoá khi vào Play từ Edit). Chạy lúc chưa Play được.
- **VERIFY Play Mode 2026-09-06**: sân thử nạp, 10 beast spawn đúng race (goblin×4/beetle×3/direwolf×3);
  đặt tên wolf thật → `tensura_direwolf→tensura_tempest_wolf`, SPD 85→149, +2 intrinsic skill, soul=Complete,
  vào party avatar; `TensuraNamingPanel.Open` dựng GO không lỗi. Tests: Body 20/20, Skill 16/16,
  Decoupling 42/42, FixedMap 42/42, Creature Body 46/46, 0 compile error.

## Bổ sung 2026-09-07 (đợt 3) — đồ thị tiến hoá điều kiện + panel v2
- **`NamedEvolutionDef` → `EvolutionPathDef`** (đồ thị phân nhánh): `fromRaceId`/`toRaceId`/`trigger`
  (v1 chỉ "naming", còn magicule_threshold/element_absorb/soul_threshold/body_transcend/mind_awaken =
  backlog) + **`conditions[]`** (`namer_ep_min`/`gender`/`pack_named`/`self_ep_min`/`soul_stock_min`/
  `namer_intent`/`has_skill`/`element_absorbed`) + `priority`. `PickEvolution(fromRace, in EvolutionContext,
  all)` — tie-break: priority → NHIỀU condition hơn (cụ thể hơn) → id. Migrate 7 file evo cũ.
- **Ý đồ = "in cảm xúc"** (lore, không phải nghề): `Intents = {"", "protective", "ambitious", "martial"}`
  (Không định hướng / Bảo bọc / Khát vọng / Thượng võ). Bỏ 4 nút job archetype cũ. skill_info
  `intentAffinity` remap warrior→martial, guardian→protective, scholar→"".
- **Panel v2**: bỏ 4 nút cứng · thêm **dòng xem trước** "Sẽ tiến hoá: X → Y" (tự suy từ giới tính +
  EP người đặt) · ô "in cảm xúc" **CHỈ hiện khi `IntentBranchOptions` > 1** (có nhánh rẽ thật theo nó).
- **Content**: race `tensura_goblina` (nữ) + `evo_goblin_goblina` [gender:1] ⟂ `evo_goblin_hobgoblin`
  [gender:0] · `evo_ogre_kijin_martial` [namer_intent:martial] làm demo nhánh rẽ cảm xúc (kijin thiên
  sát thương + Hắc Viêm). 9 evo path, `configErrors=0`.
- **VERIFY Play Mode 2026-09-07** (sân thử): goblin nam→hobgoblin, goblin nữ (Gender=1)→goblina;
  ogre + EP≥300 → panel HIỆN ô cảm xúc [neutral/martial], "martial"→kijin ATK+55%+3 skill vs
  neutral→ATK+40%+2 skill; hobgoblin EP<2500 → panel ẨN (chưa mở nhánh). Tests: Naming **28/28**,
  Skill 16/16, Body 20/20, FixedMap 42/42, Decoupling 42/42, 0 compile error.

## Nợ ngay (không chặn)
- Trigger phi-naming (`EvolutionTickSystem`) — data shape sẵn, cần code + `elementAbsorbed` tracker trên
  `TensuraBody`. Author các nhánh sâu (Goblin King, Vajra, Star/Divine Wolf, cả cây Human) = JSON sau.
- Panel IMGUI → uGUI theo bảng token (polish).
- `CorePawnInteractions` chỉ Install trong scene Wild (đúng).
- Rebirth chỉ bắt `DiscipleDiedEvent`; named BEAST chưa có tín hiệu chết chung → follow-up.
- `tensura_species_*` biome cho phép forest/rocky_mountain/plains nhưng CHƯA vào pool spawn ngẫu nhiên
  của map thường (chỉ dùng trong fixed map). Muốn quái Tensura mọc hoang → thêm vào worldgen beast pool.

## 19 quyết định người dùng đã chốt (đầy đủ trong doc §0)
author evolved race (đậm lore) · pool magicule riêng · **chỉ avatar đặt tên được** (capability
`tensura:can_name`) · giữ **mất trần magicule.maxBase VĨNH VIỄN** khi overreach · làm cả Equal + Master–Sub ·
naming môi trường/Unique/Ultimate/fusion/Awakening = **để sau** · named pawn **không permakill** (rebirth) ·
ý đồ định hình tiến hoá **v1** · named = "người của owner" (party M57 + bond) · skill v1 = Common+Extra+
Resist+Intrinsic, **dùng được ngoài combat** · mastery per-skill tái dùng `LearnedSkill` có sẵn ·
ep→combat stat **tuyến tính** (source `"tensura:ep"`, KHÔNG đụng HomProvider của xianxia) · soulIntegrity 3
mức rời · aura có v1 · ep hiển thị.

## Kiến trúc (đã đọc code, grounded)
- `TensuraBodyRegistry` keyed `PawnBody.Ref.Value` mirror `ThoughtRegistry` · `TensuraBodyIO`/`TensuraNamingIO`
  mirror `ResearchIO` (hook 3 chỗ SaveManager) · `NamedEvolutionDef : Def` (`Data/tensura_evolutions/`) ·
  `TensuraNamingRegistry` (bond forest) + `TensuraNamingOps` (thuần: Quote/ResolveNaming/PickEvolution/
  ApplyEvolution/group) · `TensuraSkillInfoDef : Def` optional metadata (list-composition) — skill THẬT vẫn
  là `SkillDefinition` JSON reuse pipeline `LearnedSkill`/`effects`/`SkillEffectRegistry` · `TensuraRebirthTracker`
  nghe `DiscipleDiedEvent` mirror GriefSystem · `TensuraContentPack.Install()` gọi từ GameRoot mirror
  `XianxiaContentPack.Install()`.
- Pack sống ở `Assets/Scripts/Tensura/` (top-level, được phép ref Sim — `Arch_CoreNeverDependsOnSim` chỉ
  guard `Scripts/Core/`).
- Evolved race JSON: goblin→hobgoblin→goblin_lord, ogre→kijin→mystic_oni, direwolf→tempest_wolf,
  lizardman→dragonewt. `PawnBody.Race` là string settable; `RaceOps` resolve raw id qua `core:` fallback.

## Nợ / rủi ro cần verify khi apply (xem APPLY_GUIDE §"RỦI RO API")
- `SaveManager` 3 điểm gọi ResearchIO — cần mở file lấy vị trí chính xác.
- `GameRoot` tên biến bus/clock lúc dựng GriefSystem.
- Named BEAST (khoá `beast:{id}`) chưa có tín hiệu chết chung → rebirth cho beast là follow-up.
- Verb "Đặt Tên" (UI) chưa viết — spec §8 + APPLY_GUIDE Bước 3.

Liên quan: [[project_m37_core_xianxia_split]] [[project_m64_combat_mode_framework]] [[project_c1_thoughts]]
[[project_m56n_race]] [[feedback_core_sim_before_xianxia]] [[reference_showdown_study]]
