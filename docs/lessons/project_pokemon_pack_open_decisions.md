---
name: project_pokemon_pack_open_decisions
description: "Pokémon content-pack — việc đã xong + quyết định thiết kế còn treo + lỗ hổng content (chốt 2026-09-10, chờ máy chính)"
metadata: 
  node_type: memory
  type: project
  originSessionId: 1241bc70-cff5-4a7d-a328-4a86b7415309
  modified: 2026-09-10T13:44:24.971Z
---

Content-pack sưu tập sinh vật kiểu Pokémon trên engine Nizhangjie, dẫn Early Access. Hệ thống ~xong, **content + vài quyết định lõi còn treo** — user đang lên kế hoạch cho lúc có máy chính.

## Đã xong (hệ thống)
- Combat P4c (`CreatureDuelCombatDriver` nhánh Pokémon: khắc hệ/STAB/crit/EV), capture tool IP-safe, lối điều khiển (Y/WASD/E/R/hotbar), team/deploy/PC, genetics (IV/EV/nature/ability), moves (4 slot + learnset + relearner), level/XP, bond.
- **Màn Summary** (`PokemonSummaryPanel`, phím B): Info/Moves/Stats, radar lục giác, party zig-zag, kéo-thả ô chiêu, nút × đóng, **Tiến hoá chủ động** (nút → menu list nhánh, xác nhận 2 bước — [[project_pokemon_controls]]).
- DevPokemon mod (149 Gen1, dev-only), UiCodeExporter + UiPanelMaterializer ([[project_ui_code_exporter]]).

## Quyết định LÕI còn treo (chặn nhiều thứ)
1. **A ⟂ B khớp thế nào** — trục A (sưu tập sinh vật) ⟂ trục B (sinh tồn + căn cứ RimWorld). 3 hình mẫu: Pokémon-forward (B tối giản) · **Palworld-forward** (sinh vật = lao động, base do sinh vật vận hành) · RimWorld-forward. Đã build nghiêng Palworld/hybrid nhưng CHƯA chốt trên giấy.
2. **Combat mode** — turn-based (`SaveData.combatModeId == "core:turnbased"` → `ArenaCombatBridge` chặn mọi va chạm Wild → Arena) là combat CHÍNH của pack? Công thức Pokémon chỉ chạy ở turn-based.
3. **Có thị trấn không** — Pokémon-forward: thị trấn = hub lõi. Palworld-forward: phụ, chủ yếu base cá nhân.
4. **Thị trấn procedural hay vẽ tay** — FixedMap M33 (ASCII) cho thị trấn "chữ ký"; city gen procedural lấp filler; có thể lai.

## Lỗ hổng loop (phân tích 2026-09-10) — "hệ có, content chưa"
- ✅ **Bắt TRONG trận** (2026-09-26): nút TÚI → chọn công cụ bắt → ném (chọn mục tiêu nếu 2v2). Engine chỉ có cơ chế + hook `CreatureDuelCombatDriver.CaptureResolver` (`TurnCaptureRequest/Outcome`, `TurnCommand.ItemId`, list `Captured`); luật bắt do `ArenaCombatBridge.ResolveCapture` cắm = CÙNG công thức ném cầu ngoài thế giới (`CorePawnInteractions.CatchChanceFor` + `FinishCapture`, lượt = vòng trận, trạng thái ×1.5). Chỉ bật khi mọi địch là quái hoang tameable. Verify Play Mode: hươu 5% máu, bóng cơ bản 17%, bắt được lần 4, thành linh thú giữ máu. User: 'code như thể nó là một cái mod'.
- ⚠ **Payoff của deploy overworld**: con theo mình làm gì ngoài lúc vào Arena? (đánh tự động / hái-khuân qua M39 capability / cưỡi). Gần như trang trí hiện tại.
- ⚠ **Áp lực lên base**: chưa có raid vào base (E5 chưa làm) → walls/defense/mood-break/temperature/bed thành code chết. Chọn: thêm nhịp quái tấn công base, HOẶC hạ kỳ vọng colony-sim.
- ⚠ **Nhịp mục tiêu phiên**: quest board + research + zone boss có mảnh, chưa thành "3 việc hôm nay" hiển thị.
- ❌ **Content gốc**: roster sinh vật trung tính, bảng type riêng, moveset, biome-vùng, boss, chuỗi tiến hoá — placeholder. DevPokemon là scaffold dev, public build xoá.

## Field Codex — ĐÃ CODE (2026-09-26)
Điểm cắm pack MỚI `Pokemon/PokemonContentPack.Install(bus)` (GameRoot 6c-quinquies, mirror Tensura) — mọi thứ Pokémon mới nên cài ở đây. Wild chỉ bắn `CreatureEncounteredEvent` (vào trận/ném cầu) + `CreatureCapturedEvent` (`CorePawnInteractions.FinishCapture`); pack nghe → `CodexRegistry` (seen/caught, khoá id thô). Danh mục = `Data/codex/*.json` (`speciesOrder` + `includeAllTameable`), `CodexOps.BuildSlots`. UI `Pokemon/UI/FieldCodexPanel` dựng code, phím N (`kb_field_codex`); lúc mở đồng bộ "đã bắt" cho linh thú đang sở hữu (`EnsureCaught`). Lưu `CodexIO` + `SaveData.codex`. Test `Run Field Codex Tests` 18. Verify Play Mode (176 loài, hươu No.064). Chưa: quest "ghi 3 loài" của Professor, icon/ảnh loài, lọc/tìm.

## NPC trainer — ĐÃ CODE (2026-09-26)
`Pokemon/Trainers/`: `TrainerDefinition` (`Data/trainers/*.json`: chance/minAge/party size/level range/speciesPool/rewardPerLevel/rematchDays/lines), `TrainerOps` (ai là trainer + đội TẤT ĐỊNH theo id cư dân, FNV không GetHashCode; pool rỗng ⇒ loài tameable có baseStats+types), `TrainerRegistry` (đã thắng ngày nào; `TrainerIO` 3 chỗ SaveManager), `TrainerBattle` (đội mang theo ≤6 ⟂ đội trainer, dùng lại driver + `ArenaBattleUI` với `FleeAllowed=false`, không bắt; máu linh thú mang vào/ghi về theo tỉ lệ; thắng ⇒ Bạc + ghi; XP CHỈ khi phe ta thắng). Nút "Thách Đấu" do pack đăng ký vào `PawnInteractionRegistry` (hiện trên dân + lữ khách qua đường chung, đi tới qua `NpcLocator.ApproachCell`). Bug thật đã sửa: `ArenaBattleUI` không bật lại menu gốc khi tới lượt mà panel vẫn Root. Đơn/đôi: `TrainerBattle.ResolveActivePerSide` = `TrainerDefinition.activePerSide` (0 ⇒ chế độ arena chung `ArenaCombatBridge.ResolveActivePerSide()` = `core:turnbased.activeSlotsPerSide`), kẹp 1..2 và không vượt số con mỗi phe; kiểu mẫu `trainer_duo` luôn đánh đôi. Scene `CombatArena.unity` chỉ là bàn thử, game thật dựng UI tại chỗ. Test `Run Trainer Tests` 18. Verify Play: sect_11 có 4/66 trainer, thua + thắng (200 Bạc) + trận đôi Lâm Bất Hậu (chọn mục tiêu, 300 Bạc). Chưa: dấu hiệu nhìn thấy trainer trên map, AI trainer chủ động thách, gym/leader.

## Tiến hoá đầy đủ + Shiny — ĐÃ CODE (2026-09-26)
Tiến hoá VẪN do người chơi bấm (chốt cũ giữ). `EvolutionPath.conditions` [{id,value}] + `EvolutionConditionRegistry` (lõi: min_level/min_bond/time_of_day/weather/biome/knows_move/gender; mod `Register`; id lạ ⇒ KHOÁ). `requires:"item"` nay cần vật phẩm thật + TIÊU 1 khi tiến hoá. Bối cảnh gom ở `CompanionOps.BuildEvolutionContext` (Wild), `EvolutionOptions`/`CanEvolveNow`/`EvolveChecked` (Summary dùng). Bug thật đã sửa: nhiều nhánh cùng đích ⇒ lấy nhánh ĐỦ điều kiện. Arena báo "X có thể tiến hoá!" cho con VỪA đủ (chỉ khi linh thú dự trận — hiện trận Wild phe ta là pawn nên ít khi kích). Shiny: `CreatureGenetics.shiny` roll SAU CÙNG trong `Roll` (không xê dịch IV cũ) theo `Data/genetics/shiny_config.json` (1/4096); lưu qua genetics save; `ShinyOps.ColorFor` (species.shinyColorHex hoặc xoay hue) cho token quái/linh thú, tên vàng "(Shiny)", pill Summary, `CodexEntry.shinyCaught`. Test `Run Evolution Shiny Tests` 22. ⚠ Test phải NẠP LẠI DataRegistry sau khi Clear (đã làm Duel đỏ).

## Human companion — ĐÃ CODE v1 (2026-09-10)
5 file mới + wire GameRoot §8-bis. **Tag:** `Core/Pawns/Party/CompanionTagState.cs` — static HashSet `PawnRef.Value`, session-only (mirror `CreatureDeployState`), companion người nào "ra sân đánh cùng". **Farm order:** `World/Wild/Companions/` — `CompanionFarmOrder` + `CompanionFarmRegistry` (ServiceLocator, KHÔNG persist — như job dở) + `CompanionFarmOps` (THUẦN: `StartFarm`/`CancelFarm`/`ResumeFarm`/`Tick`) + `CompanionFarmSystem` (MonoBehaviour trên `~GameRoot`, `GameClock.OnSlowTick`, mirror `AbsentPartyNeedsSystem`). Mỗi slow-tick: 1 trận headless `CreatureDuelCombatDriver` (đội creature companion `ActivePerSide=1` vs 1 con hoang `MakePokemonCombatant`) → `while(IsRunning) AdvanceRound()` → `CreatureDuelRewards.GrantXp(Team.A)` (XP vào con DỰ TRẬN). **Fuel** = 1 số cấp-LỆNH, trừ `fracLost*0.25 + downed*0.15 + (thua?0.05)` mỗi trận; ≤0.15 → tìm party-mate `core:medical` → `CreatureCareOps.HealParty` + fuel=1; không có → Paused ("ghé Care Center"). Đạt `targetLevel` → Done. **Giới hạn v1 (user sẽ căn):** đối thủ bịa từ pool loài `species_*` Pokémon-model (chưa theo biome/leash — worldX/Y chỉ lưu); KHÔNG persist HP/PP từng con giữa trận (mỗi trận đội đầy máu, fuel là proxy hao mòn); tỉ lệ thua + tốc fuel drain hiện cao với mon cấp thấp 2 chiêu. Test `Run Companion Farm Tests` (18). **VERIFY PLAY MODE 2026-09-10** (CityScene, đủ 5 nhánh: tag/loop/XP-participants/fuel-pause/medic-autoheal/target-Done). **CHƯA:** UI ra lệnh (nút ở cột party/switch-move màn Summary), hook vào trận thật (tag → creature team companion cũng ra sân 2v2), reframe recruit-via-arc, `CompanionArcDefinition` vs M62.

## Human companion — đồng hành kiểu anime (hướng, 2026-09-10)
Chỗ project ăn đứt game gốc: engine có pawn xã hội đầy đủ (M28 recruit · M57 party · M29b opinion · M48-54 chronicle→ký ức→nhịp thoại→đại từ · M35 grief · M39 capability · M59 travel). Companion người = `PawnBody` trong `PawnPartyRegistry`, có **đội quái RIÊNG** (thêm owner key trong `CreatureTeamRegistry` — đã support), 1 capability định danh (Cook/Medic/Breeder/Ranger), traits/skills/needs, 1 arc, bond track = dùng lại `SocialRegistry` opinion.
- **CHỐT: không cap số companion người** — chỉ cần "tag" state ai đang ra sân đánh cùng (mirror `CreatureDeployState` HashSet, nhiều-out). Battle = `CreatureDuelCombatDriver` `ActivePerSide=2` (2v2 tag, đã có).
- **CHỐT: lệnh "farm cấp độ"** — player ra order tại vùng → companion vào vòng lặp JobGiver: tìm quái hoang trong leash → duel turn-based AI-vs-AI (`PickMove` đã có, driver terminate có winner) → XP → lặp; dừng khi PP/HP cạn / hết leash / đạt cấp mục tiêu / huỷ. Có Medic đi cùng → auto-heal giữa trận. Chạy NỀN (mirror `AbsentPartyNeedsSystem` M59), báo tổng khi xong.
- Nhập đội: KHÔNG "trả Bạc ở quầy" (M28 `NpcRecruitOps` cần reframe) — gặp trên đường (`evt_traveler_encounter`) / móc arc (NPC town có vấn đề) / kết thân tới ngưỡng opinion / rival hoá bạn.
- Rời đội: arc xong → rời (bittersweet, giữ quan hệ, tái xuất lúc khẩn cấp) HOẶC xung đột (opinion tụt). Roster tự xoay theo region.
- **CHỐT (2026-09-10): XP kiểu Pokémon thường** — con nào DỰ TRẬN (ra sân ≥1 lần) mới nhận XP. Cài xong: `TurnCombatant.Participated` (set ở `CreatureDuelCombatDriver.FireEnterBattle` — active đầu trận + mỗi switch-in + mọi con ở FFX), `CreatureDuelRewards.GrantXp` lọc `c.Participated` (bond + EV cũng gate theo). Con ngồi băng ghế suốt = 0 XP. Test `Run M64 TurnBased Tests` block "Chia XP kiểu Pokémon" (247/247). Companion người KHÔNG bắt buộc để farm ("bạn đồng hành, không phải công cụ" — player tự farm được).
- **CHƯA chốt**: exp-share sang quái player bench (hiện KHÔNG có) · Arc = `CompanionArcDefinition` riêng vs quest chain M62 scope `companion:{id}`.

## Từ vựng IP-safe (English — id/role/displayName đều tiếng Anh, chỉ comment tiếng Việt)
Care Center (`care_center`, heal+storage) · Supply Store (`supply_store`, role shop) · Battle Hall (`battle_hall`) + Hall Master + Seal (badge) · Nursery (`nursery`, breeding) · Skill Hall (`skill_hall`, move tutor/reminder) · Skill Workshop (`skill_workshop`, craft "skill disc" = TM, dùng M26 WorkOrder + `CreatureMovesRegistry.TeachMove`) · Field Codex (dex) · Proving Grounds (`proving_grounds`, battle tower) · Champions' Hall (`champions_hall`, league) · Grand Store (`grand_store`, department store) · Research Lab (`research_lab`, professor → starter + codex).
⚠ File `pk_*` mình vừa viết (2026-09-10) để displayName TIẾNG VIỆT — cần đổi sang English + đổi id `pk_creature_center`→`care_center`, `pk_mart`→`supply_store`, `pk_gym`→`battle_hall`.

## Starter town (grounded) — gộp "làng nhà" + "hub đầu tiên"
Mọi town: **Research Lab** (Professor → starter + Field Codex + quest "log 3 loài") · **Care Center** (heal + storage) · **Supply Store** (capture tool/potion) · **Grocer/market** (đồ ăn thật — hệ đói D1/D2) · **Inn** (Rest, phòng người chơi = spawn) · 3-4 **houses** (1 tặng tip, 1 trẻ con cho trận tập) · **Town gate** + **Notice board** (quest). KHÔNG có Battle Hall/Nursery/Champions' Hall (ở hub sau). Thị trấn nhỏ nên dùng **FixedMap M33 vẽ tay**, không procedural (procedural chỉ hợp thành LỚN).
Opening "cơ bản" = **Field Assistant** scenario (`StartScenarioDefinition`, chọn lúc new-game như `combatModeId`): Professor thiếu người → nhờ lập danh mục → cho starter + 3 tool + Codex → bắt/ghi 3 loài ở vùng cạnh town → về báo → unlock. Kiểu khác (lang bạt nhặt thú / học việc thương nhân / tân binh kiểm lâm) = scenario khác, cùng cơ chế.
Vì NPC ở đây là pawn THẬT (opinion/ký ức/chiêu mộ) → **không cap số nhà/dân**; town đông = pool chiêu mộ + texture quan hệ. 2 vòng "đội" song song đổ về party: bắt quái ⟂ kết thân/chiêu mộ NGƯỜI. Civic buildings phục vụ vòng 2: Tavern/Inn (gặp lữ khách, nối `evt_traveler_encounter`) · Notice board (việc → tiếp xúc NPC) · Guild/Hall (chiêu mộ chuyên gia) · Workshops (NPC hành nghề).

## City auto-gen — thuật toán ỔN, content SAI chủ đề (2026-09-10)
Hệ gen (BSP/trục/hẻm/nhà dãy/nội thất theo vai phòng/tường-cửa/NpcBrain) generic, chạy tốt. **Data feed vào toàn xianxia** (nhà tông môn, cư dân "đệ tử", Linh Thạch, tên Hán, thoại ta/ngươi) — đúng CLAUDE.md §5.8. Sửa = viết bộ data city mới song song, KHÔNG đụng hệ. Thị trấn Pokémon cần:
- **Trung tâm hồi phục** (heal đội + hồi PP + truy cập PC) = building quan trọng nhất, CHƯA có. Nhỏ: `BuildingDefinition` + handler `core:creature_heal` mirror `core:storage`.
- Cửa hàng bán capture tool / thuốc / held item (khung M65 có, cần catalog trung tính).
- Nhà + cư dân dân-thường/huấn-luyện-viên (không phải đệ tử), tên trung tính, currency chỉ Bạc.
- Gym / sảnh khiêu chiến (tuỳ), boss trainer.

## Menu — nợ nhỏ (khi có máy chính)
Căn layout tinh, render model 3D ô chân dung, Held Item thật, cuộn khi list cột phải tràn, nút reorder đội, dex number, tiến hoá tiêu đá thật.

Liên quan: [[project_pokemon_controls]], [[project_m64_combat_mode_framework]], [[project_capture_tool]], [[project_core_content_pack]], [[project_citygen_wallcity]].
