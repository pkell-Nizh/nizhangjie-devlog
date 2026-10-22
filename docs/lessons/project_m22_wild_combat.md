---
name: project-m22-wild-combat
description: "M22 Combat thật trên Wild map (HP/damage/chết/loot) — Phase 1 ĐÃ CODE 2026-07-27, ĐÃ VERIFY Play Mode 2026-08-09 sau 2 vòng fix (quái chủ động đánh + skill nhắm đất)"
metadata:
  node_type: memory
  type: project
  originSessionId: 3baff345-b610-4956-8004-5715cbd5406e
  modified: 2026-08-09T11:22:02.856Z
---

**Phát hiện 2026-08-09:** khi bàn "làm content cơ bản gì tiếp theo", đề xuất ban đầu là "wire combat thật
vào Wild" tưởng là việc MỚI hoàn toàn (dựa trên [[project_m21_threat_reaction]] nói combat thật "milestone
riêng, chưa code"). Đọc thẳng `Assets/Scripts/Nizhangjie/M22_WILD_COMBAT_DESIGN.md` mới biết: **đã CODE
xong Phase 1 từ 2026-07-27**, chỉ chưa verify Play Mode — memory không hề có entry cho M22, nên bị bỏ sót.
**Bài học:** trước khi đề xuất "xây tính năng X", LUÔN grep/ls `Assets/Scripts/Nizhangjie/*.md` tìm doc Mxx
liên quan — memory có thể thiếu milestone dù code đã có.

**Quyết định kiến trúc lớn (đã chốt, đọc doc gốc trước khi đổi):** KHÔNG bê `CombatEncounter`/`CombatGrid`/
`GridMovementSystem`/`CombatBehaviorSystem`/`CombatRTwPController` (scene `combat_in_prototype`) sang Wild —
Wild đã có `GridSystem`/`RegionPathfinder`(M14)/`PawnAiBrain`/`BeastBehaviour` riêng, bê nguyên = 2 nguồn sự
thật tranh nhau. Chỉ tái dùng LÕI TÍNH TOÁN: `DamageCalculator`, `CharacterStats`, `CombatUnitDefinition`,
`CombatUnit` (chỉ làm vỏ mang Stats/Shield), toàn bộ `IHitShape`(5 shape)+`AoeDamageResolver`.

**Kiến trúc 3 lớp mỏng (`World/Wild/Combat/`):**
- `IWildCombatant` (bridge) — 2 impl `PawnCombatant`(bọc Disciple)/`BeastCombatant`(bọc BeastActor). **HP THẬT
  nằm ở entity gốc** (Disciple.CurrentHP / BeastActor.hp), KHÔNG ở CombatUnit — vỏ CombatUnit bị dựng MỚI mỗi
  lần Refresh() nên KHÔNG được dùng làm nguồn HP (bẫy đã ghi rõ trong doc §2.4).
- `WildCombatOps.cs` (plain C#, test được) — `ResolveAttack`→DamageCalculator→trừ HP thật; `AttackInterval`
  suy từ SPD kẹp [0.6s,3s]; `HandleBeastDeath`→rớt loot qua `WorldItemDrop` (M18, nhặt bằng PickUp).
- `WildCombatGrid.cs` + `WildSkillOps.cs` — tái dùng NGUYÊN 5 `IHitShape` qua đúng 2 hàm `CombatGrid.InBounds`/
  `IsTerrainBlocked` (field mới `CombatGrid.ExternalTerrainBlocked` uỷ quyền sang `GridSystem.IsPassable`,
  KHÔNG copy ô nào → tường M15 tự có hiệu lực). `WildCombatGrid.Refresh()` quét lại occupancy mỗi lần thi
  triển thay vì giữ đồng bộ tăng dần.
- Ai ra lệnh đánh: verb **"Attack"** (Tier 0 ForceJob, player right-click quái) · `JobGiver_MeleeAttack`
  (**Tier 2, ĐĂNG KÝ TRƯỚC `JobGiver_Threat`** — cùng tier, OrderBy stable giữ thứ tự add, đảo thứ tự = pawn
  bỏ chạy dù quái đã cắn vào mặt) · `RetaliateGoal` (BeastGoals, quái bị đánh thì đuổi+đánh trả kể cả loài
  passive).

**Chết:** quái hp≤0 → Dead + BeastSpawner.Despawn + rớt loot. Pawn CurrentHP≤0 → **KHÔNG permadeath (v1)**,
`Status="Downed"`, PawnAiBrain bỏ qua pawn downed. Hồi sinh/y tế = phase sau.

**§4 CỐ Ý CHƯA LÀM (đọc kỹ trước khi tưởng là thiếu sót — đây là danh sách việc còn lại thật sự cho content
combat, không phải khởi tạo từ 0):**
1. ATB gauge/pause-lite FF7R/RTwP — nhịp hiện = cooldown SPD + cast time, gauge chỉ có nghĩa khi cần xếp
   hàng lệnh.
2. **Ailment/elemental STATE** — `AoeDamageResolver` CÓ gọi `ElementalReaction.ApplyElementalHit` (damage
   nguyên tố CÓ tác dụng), nhưng aura/stack đọng trên VỎ CombatUnit bị vứt mỗi Refresh() → phản ứng/hiệu ứng
   KÉO DÀI chưa hoạt động. Cần `AilmentController` sống theo entity wild + tick riêng.
3. Knockback — `SkillDefinition.knockbackTier/Distance` chưa đọc.
4. Footprint quái nhiều ô — `WildCombatGrid` đăng ký 1 ô/entity, quái to bị AOE tính thiếu ô.
5. Quái chủ động săn pawn (aggro) — `ApproachPlayerGoal._enableAggro` mặc định FALSE (cố ý, bật trước khi
   cân bằng combat = pawn chết oan).
6. Damage number nổi — polish.
7. Pawn chết thật (permadeath) — xem mục "Chết".

**Test:** `Tests/Editor/WildCombatTest.cs` menu **Nizhangjie → Run WildCombat Tests** (plain C#, resolve
damage/crit/lifesteal/chết đúng ngưỡng/InRange Chebyshev/loot rơi đúng ô/JobGiver_MeleeAttack chỉ nhận việc
khi quái nguy hiểm ĐANG kề).

**ĐÃ VERIFY Play Mode 2026-08-14 (user xác nhận) (việc đầu tiên khi quay lại M22):** right-click quái → Attack → pawn đi tới, đánh,
quái đánh trả, quái chết → rớt đồ → nhặt bằng PickUp. Không tự động test được (cần token/tick thật).

**Verify Play Mode đợt 1 (2026-08-09):** user báo 3 điều:
1. Quái Wild map hiện KHÔNG chủ động tấn công pawn → không test được nhánh "tự vệ" của Tier 2. Root cause
   ĐÚNG như dự đoán trong bảng nợ thiết kế: `BeastBehaviour._enableAggro=false` mặc định (gate
   `ApproachPlayerGoal`) — KHÔNG PHẢI bug, là thiết kế cố ý "tránh dồn player khi chưa có combat cân bằng".
2. Pawn vẫn Flee đúng khi ở gần quái (M21 không hồi quy) — xác nhận tốt.
3. Cast skill "không được" → user xin thêm highlight tô theo màu hệ + dùng làm debug. **ĐÃ CODE ngay trong
   phiên:** `WildSkillOps.PreviewCells(skill, casterCell, targetCell, view)` (share đúng logic hình học với
   `Cast`, KHÔNG áp damage) + `WildTilemapRenderer.HighlightCells(cells, color)` (mirror `HighlightRect`,
   nhận thẳng list ô) + `WildSceneInteraction.ShowSkillPreview` (gọi mỗi frame khi `_pendingSkill != null`,
   màu = `CombatSkillSlotUI.AccentColor(skill.elementAffinity)` alpha 0.5, lazy `_previewGrid` reset ở
   `OnGenerated`). Test `WildCombat Tests` vẫn 38/0 sau khi thêm.

**Research qua Unity MCP (execute_code) lộ ra 2 điều SỬA LẠI giả thuyết ban đầu — đọc kỹ trước khi tưởng
UI "chưa wire":**
- `WildCombatHotbarUI` GameObject **CÓ THẬT** trong `WildLocal.unity` (`HUD/.../CombatHotbarUI`), field
  `_interaction`/`_group`/`_root`/`_bookIcon`/`_slotPrefab`/`_slotContainer` đều wire đúng. `BtnDraft`
  (`PawnBar/Actions/BtnDraft`) → `PawnBarController.OnDraftToggleButton` → `WildSceneInteraction.
  ToggleDraftForSelected` — ĐÃ WIRE ĐÚNG (comment code còn ghi rõ: từng wire NHẦM sang
  `CombatCoreDriver.UiToggleDraft` cũ, ĐÃ SỬA từ trước). 2 nút mũi tên đổi bộ công pháp cũng wire đúng
  `OnBookNext`/`OnBookPrev`. → Root cause "skill cast không được" nhiều khả năng KHÔNG PHẢI thiếu UI, mà là
  thiếu phản hồi hình ảnh (đúng như user nghi — không có gì để "thấy" là đã cast) — preview highlight mới
  thêm nên xác nhận được.
- Chỉ riêng `_bookNameText` null (thiếu label tên bộ công pháp) — cosmetic, chưa sửa.

**GOTCHA quy trình (tự mắc lỗi, ghi lại):** lúc research thấy `SlotContainer` có sẵn 3 `SkillSlot(Clone)`,
suy luận nhầm theo pattern đã biết ([[project_m26_production]] "Destroy() Edit Mode hoãn tới cuối frame,
SaveScene chạy trước ⇒ clone dính vĩnh viễn") rồi `DestroyImmediate` chúng — **KHÔNG kiểm tra Play Mode
trước**. Gọi `EditorSceneManager.MarkSceneDirty` báo lỗi "cannot be used during play mode" mới lộ ra: Editor
đang **is_playing=true** (user đang test tay giữa chừng) — 3 clone đó là UI runtime THẬT của phiên Play đang
chạy (pawn đã draft + có 3 skill), không phải rác kẹt trong file scene. Đã xoá nhầm 3 slot đang hiển thị của
phiên Play (tự phục hồi — `DestroySpawnedSlots()` có guard `if (s != null)` nên lần `Refresh()` kế tiếp tự
dựng lại sạch, và Play Mode không lưu xuống file nên không hư gì vĩnh viễn) nhưng gây gián đoạn phiên test
của user. **Luật: LUÔN check `editor_state.data.editor.play_mode.is_playing` (`ReadMcpResourceTool
mcpforunity://editor/state`) TRƯỚC khi diễn giải state đọc qua MCP là "đã lưu trong scene" — đọc component
qua MCP phản ánh runtime hiện tại, không phân biệt Play vs Edit mode tự động.**

**Verify đợt 2 (2026-08-09, cùng ngày) — user báo 2 lỗi THẬT, đã sửa cả 2 (test 38/0 + 6/0 xanh, CHƯA
verify Play Mode):**

**Lỗi 1: "quái aggressive không hề chủ động tấn công dù pawn ở cực kỳ gần".** Root cause SÂU HƠN cờ
`_enableAggro` — `ApproachPlayerGoal` (priority 90) có **3 tầng hỏng cộng dồn**, đều là tàn dư thời chưa có
combat:
  1. Nó CHỈ trả `BeastGoalKind.Approach` rồi đứng chờ ở `_holdDist` — **không bao giờ đánh**, kể cả khi bật
     cờ (comment gốc ghi thẳng "combat chưa wire"). Nghĩa là bật `_enableAggro` cũng vô ích, quái chỉ tới
     đứng nhìn. `BeastGoalKind.Attack` (`TryApplyAttack` lo áp sát + nhịp SPD + damage + pawn gục) CHỈ được
     `RetaliateGoal` phát ra ⇒ quái đánh trả thì có, chủ động thì không.
  2. Đọc `ctx.PlayerCell` = 1 token cố định → 3/4 pawn party "vô hình" (đúng lỗ hổng M21b đã vá cho
     `FleeThreatGoal` nhưng CỐ Ý bỏ qua chỗ này vì hồi đó aggro vẫn tắt).
  3. Chỉ khớp `"aggressive"`, **bỏ sót `"territorial"`** — trong khi `JobGiver_MeleeAttack.IsDangerous` luôn
     coi CẢ HAI là nguy hiểm ⇒ pawn tự vệ trước territorial mà territorial không bao giờ đánh: 2 đầu của
     cùng 1 luật lệch nhau.
  **Fix:** `ApproachPlayerGoal` → viết lại thành **`HuntPawnGoal`** (`BeastGoals.cs`): trả
  `BeastGoalKind.Attack` nhắm pawn GẦN NHẤT trong `ctx.PawnCells` (tất cả pawn), aggressive săn bất kỳ ai
  trong `_senseRange`, territorial chỉ đánh kẻ ĐANG Ở trong `wanderRadius` quanh `self.Home` (đúng nghĩa
  "territorial"). `_holdDist` GỠ luôn (khái niệm "áp sát rồi đứng chờ" chết theo).

  **Chỗ đặt cờ ĐỔI CÓ CHỦ ĐÍCH — chống bẫy serialize:** `BeastBehaviour._enableAggro` đã bị Unity serialize
  `false` vào `WildLocal.unity`, nên **đổi default trong code sẽ KHÔNG có tác dụng** (đúng bẫy đã dính ở
  beast density [[project_beast_identity]]). Giải: GỠ HẲN field cũ, thêm field MỚI `_beastsHuntPawns = true`
  trên `WildCombatSystem` → `WildCombatConfig.BeastsHuntPawns`. **Field MỚI trên component CÓ SẴN thì tự lấy
  default từ code** — không cần user vào Inspector sửa tay. Đây cũng là chỗ ở đúng: mọi knob cân bằng combat
  (leash/grudge/nhịp đánh/AOE) đều đã ở `WildCombatSystem`.

**Lỗi 2: "skill click lên tiles không dùng được".** Root cause: `WildSceneInteraction` right-click khi có
`_pendingSkill` CHỈ cast nếu trúng `InteractableKind.Beast`; bấm xuống ô đất thì skill bị huỷ **IM LẶNG**
rồi rơi xuống mở context menu như thường. Nhưng skill AOE `targeting_mode: target_point` (rect/cross/circle
quanh ô click) VỐN sinh ra để nhắm ĐẤT — đó là cách dùng chính, không phải ngoại lệ.
  **Fix:** `JobGiver_MeleeAttack.BuildSkillJobAtCell(pawn, cell, skill, data, spawner)` (mới) + tách ruột
  chung `BuildSkillJobCore(pawn, Func<Vector2Int> aimCell, BeastActor target, ...)` — 2 nhánh khác đúng 2
  chỗ: ô nhắm ĐỘNG (bám quái đang chạy) vs CỐ ĐỊNH, và `FailWhen(!target.Alive)` chỉ gắn khi có quái. Input:
  right-click giờ LUÔN thi triển (trúng quái → nhắm con đó, trượt → nhắm đất). `WildSceneInteraction.
  DispatchSkill` gom phần chung (tra def/kiểm pawn/hồi chiêu/ForceJob) và **log rõ MỌI đường thoát**.
  **Hệ quả phải xử lý:** right-click không còn là đường huỷ chiêu → thêm **ESC** để bỏ chiêu đang cầm.

**Verify Play Mode ĐẠT (2026-08-09, cùng ngày, sau khi sửa cả 2 lỗi trên):** user báo "test pass rồi" — quái
aggressive/territorial giờ chủ động lao vào đánh, skill cast được cả lên quái lẫn xuống ô đất, preview màu hệ
hiện đúng. Combat thật trên Wild coi như ĐÃ CHƠI ĐƯỢC (đúng chuẩn [[feedback_done_means_playable]], không chỉ
test xanh).

**ĐỢT 3 (2026-08-09, cùng ngày) — user chốt "code hết đi trừ Ailment/Elemental kéo dài" + "sửa data làm
pawn yếu lại nhiều để test". 4 mục §4 + 1 đợt cân bằng. Test `WildCombat` 53/0 (từ 38, +15 assert),
`JobGiverThreat` 6/0, `PawnAiBrain` 4/0, `JobDriver` 16/0. ĐÃ VERIFY Play Mode 2026-08-14 (user xác nhận).** Chi tiết đầy đủ ở doc
`M22_WILD_COMBAT_DESIGN.md` §7; những điều KHÔNG suy ra được từ code:

- **`JobGiver_Rescue` (Tier 2, add SAU MeleeAttack+Threat)** — pawn gục được đồng đội tới sơ cứu, tỉnh với
  `RescueHealFraction` máu. **Phạm vi cố ý hẹp: KHÔNG phải hệ y tế** (thương tật/giường bệnh là việc riêng),
  không thêm khái niệm mới nào. `WildPlayerToken.RescueClaimed` mirror `Designation.Claimed` chống cả party
  xúm vào 1 người. **Bẫy:** phải reset `Status` khi cứu xong chứ không chỉ cộng máu — `PawnAiBrain` dùng
  chính `Status == DownedStatus` làm mốc "đã xử lý cú gục này", để nguyên thì lần gục SAU không huỷ được job
  của nạn nhân (bug im lặng, chỉ lộ ở lần gục thứ hai).
- **`WildKnockback`** — KHÔNG port `KnockbackSystem` arena (nó dời vị trí qua `CombatGrid.PlaceUnit` +
  `CombatUnit.Movement.SetPosition` + `CombatEncounter.GetUnit`, cả 3 là bộ máy arena). **Luật vàng khi dời
  vị trí entity Wild: phải làm ĐỦ 3 việc — ô logic + vị trí TOKEN + XOÁ PATH đang đi.** Thiếu token ⇒ con vật
  đứng nguyên trên màn hình mà logic tưởng đã dời; giữ path ⇒ `BeastBehaviour.StepAlong` lết nó NGƯỢC VỀ chỗ
  vừa bị hất khỏi. Nên mới sinh `WildPlayerToken.ForceCell` + `BeastSpawner.Displace` (spawner sở hữu
  token/view của quái nên chỉ chỗ đó làm đủ 3 việc được).
- **Footprint nhiều ô** — `WildCombatGrid.Register` đọc `UnitFactory.FootprintOffsets` (CÙNG bảng arena dùng
  đặt golem 2×2 ⇒ 2 nơi không lệch được). Ô ngoài biên **lọc bỏ** chứ không kẹp: `PlaceFootprint` validate cả
  cụm và từ chối TOÀN BỘ nếu 1 ô hỏng ⇒ quái to sát mép map sẽ tàng hình hoàn toàn.
- **`WildCombatFx`** (số damage nổi) — world-space dựng bằng CODE, **no-op khi chưa `Bind`** (mirror
  `WildCombatDeath`) nên gọi thẳng từ `WildCombatOps`/`WildSkillOps` (plain C#) vẫn không đẻ GameObject lúc
  chạy Editor test. Màu: đỏ khi PAWN ăn đòn. Dùng `UnityEngine.Time` không phải `SimTime` (trình diễn, M23.1).
  Ô hiện số chụp TRƯỚC knockback.
- **Cân bằng pawn (§7.5) — quyết định quan trọng, đừng "sửa" lại:** pawn cũ gần bất tử (840-1120 HP vs sói
  110 HP) vì `MaxHP = Căn cốt×4 × Hom`, Hom=3.5 ở Luyện Khí tầng 3. Sửa **thuần data** bằng `homOverride: 1`
  + `statOverrides` (2 field đã có sẵn trong `DiscipleDefinition`). **KHÔNG hạ 6 core stat** dù đó là cách
  hiển nhiên nhất — core stat lái CẢ combat LẪN năng suất làm việc (`(Agility+Foundation)/200×Loyalty/100`),
  hạ Căn cốt để bớt máu sẽ làm pawn chặt cây/xây nhà chậm hẳn ⇒ sửa cân bằng combat lại phá cân bằng kinh tế.
  **Thân pháp giữ nguyên** vì nó lái SPD = nhịp đánh (hạ xuống là pawn chạm sàn 3s/đòn còn sói 1.28s).
  Số mới: Cam 110HP/28ATK/14DEF · Vàng 150/20/22 · Đen 115/25/15 · Lam 85/18/10 → trận với Hôi lang thành
  thắng sát nút, Lam đơn độc thua, Nâu hùng 1v1 thắng Cam.

Liên quan: [[project_combat_redesign]] (lõi tính toán C1-C10 tái dùng), [[project_m21_threat_reaction]]
(Tier 2 Flee — giờ đứng CHUNG tier với JobGiver_MeleeAttack, thứ tự add quyết định né hay đánh),
[[project_pawn_ai_brain]] (5-tier brain), [[project_beast_identity]] (BeastActor/Provider dùng ở đây),
[[feedback_core_sim_before_xianxia]] (combat generic = đúng hướng ưu tiên).
