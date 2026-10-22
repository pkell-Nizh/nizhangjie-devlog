---
name: project-m37-core-xianxia-split
description: "M37 Tách Core Life Sim ⟂ Xianxia — PawnBody là nhân vật lõi, Disciple thành lớp con; tu tiên thành DỊCH VỤ gỡ được; hệ mới PHẢI nhận PawnBody"
metadata: 
  node_type: memory
  type: project
  modified: 2026-08-15T09:59:24.113Z
  originSessionId: aff26471-c04e-4550-8ceb-471cf122f948
---

**Luật một câu (quan trọng nhất file này): hệ thống MỚI nhận `Core.Pawns.PawnBody`, KHÔNG nhận
`Disciple`.** Cần tu vi thì *hỏi* `Sim.Xianxia.CultivationRegistry`, đừng *đòi* kiểu.

**Yêu cầu gốc của user (2026-08-15):** "avatar player, các pawn và npc không bị phụ thuộc vào disciple,
mà hệ thống disciple mới là thứ nhận và trả thông tin lại cho họ" + "xianxia chỉ là một nhà cung cấp
'dịch vụ game'" + "không làm mất đi toàn bộ chức năng của một thế giới bình thường khi không có xianxia".

**Khảo sát trước khi làm (số liệu thật, không đoán):** `Disciple` xuất hiện ở **199 file**, `new
Disciple(...)` ở **35 file**, còn `PawnRef` (tầng trừu tượng M28) chỉ **28 file** — M28 chỉ landed ở tầng
DANH TÍNH, chưa chạm tầng THÂN XÁC. Ba mắt xích khoá cứng thật: `WildPlayerToken.Unit` alias thẳng tới
`Disciple`; `PawnAiContext.Pawn` là `Disciple` ⇒ **cả 17 JobGiver** khoá theo; `Hunger/Rest/Mood/CurrentHP`
là **field trên `Disciple`** ⇒ thú/NPC vĩnh viễn không biết đói/mệt/suy sụp. Bằng chứng của cái giá:
Companion (M29) + Beast đã phải **fork hẳn đường ống AI riêng** để né `Disciple`.

**Kiến trúc mới — 3 mảnh:**
1. `Core/Pawns/PawnBody.cs` — thân xác sống tổng quát. **`Disciple` nay là LỚP CON** ⇒ 199 file cũ biên
   dịch nguyên vẹn, không phải sửa.
2. `Sim/Xianxia/CultivationRegistry` + `CultivationProfile` — sổ tu vi keyed `PawnRef`. **`ProfileOf()`
   trả `null` cho phàm nhân và đó là HỢP LỆ.** `Disciple.Realm/QiLayer/...` nay chỉ là **proxy**, dữ liệu
   KHÔNG còn trên object (đã verify: ghi `d.Realm` xong đọc `ProfileOf(d.Ref).Realm` ra đúng).
3. `Sim/Xianxia/XianxiaContentPack.Install()` — điểm cắm DUY NHẤT, ở `GameRoot` mục 6c-ter. `PawnBody`
   khai sẵn 2 chỗ trống `HomProvider`/`HomApplier` và KHÔNG tự điền.

**PHÉP THỬ SỐNG-HAY-CHẾT:** bỏ `XianxiaContentPack.Install()` thì game vẫn chạy đủ (đói/mệt/suy sụp/làm
việc/đánh nhau/chết), chỉ là ai cũng `Hom = 1`. Test `Run Pawn Decoupling Tests` §6 khoá điều khoản này.
Ngày nào nó đỏ = có người kéo tu tiên ngược vào lõi.

**Ranh giới đã chốt (bảng đầy đủ ở CLAUDE.md §5.2):** CORE = danh tính/HP/đói/mệt/tinh thần/túi đồ/trait/
skill-cơ-chế/capability/lịch/job/chế-tác-cơ-chế/sức khoẻ/xã hội/`Hom` cơ chế. XIANXIA = Cảnh Giới/Linh
Căn/thiên kiếp/công thức Hom-từ-cảnh-giới/tông môn-lương-thăng cấp/nội dung JSON (công pháp, đan dược, ô
`xianxia:artifact`).

**Quyết định đáng nhớ:**
- **`PawnRef` GIỮ tiền tố `"disciple:"`** — là ĐỊNH DẠNG DÂY đã vào save + combat `unitId`; ở tầng mới
  nghĩa là "pawn bền do player sở hữu", không phải "phải tu tiên". Đổi = vỡ mọi save.
- **`DiscipleRegistry` nay giữ `PawnBody`** (tên cũ, đổi vai). Hệ tông môn (lương/thăng cấp/nhiệm vụ/
  luyện đan/rèn) lọc qua view có kiểu `AsDisciples()`/`GetDisciple()`/`AliveDisciples()` thay vì nới ra.
- **Chế tác/trang bị/job là CƠ CHẾ LÕI, nội dung mới là xianxia** — nới `CraftingEngine`/`RecipeRegistry`/
  `EquipmentManager`, giữ JSON.
- `Loyalty` ⟂ `Mood` **cố ý không cộng dồn** (Loyalty = thái độ với TẬP THỂ, Mood = cá nhân).
- UI đọc tu vi qua `PawnCultivationExtensions`, và **tự ẩn dòng Cảnh Giới/Linh Căn** khi pawn phàm nhân.

**⚠ BẪY POWERSHELL — dính thật trong chính đợt này, sẽ dính lại nếu quên:**
`-replace` mặc định **KHÔNG phân biệt hoa thường** ⇒ sweep `\(Disciple ` nuốt luôn `(disciple == null`
thành `(PawnBody == null` (5 chỗ, chỉ lộ lúc biên dịch). Sweep đổi tên kiểu C# **PHẢI dùng `-creplace`**.
Và `\bDisciple\b` còn khớp 3 lớp khác nhau ngoài KIỂU: **segment namespace**
(`Nizhangjie.Sim.Disciples.Disciple`, thư mục `UI/Sim/Disciple` → gãy `using`), **tên thành viên**
(`.Disciple` property của `CharacterViewCardUI`/`ItemViewContext`), và **field của event**
(`e.DiscipleId` của `DiscipleDiedEvent`). Mỗi lớp im lặng cho tới compile. Xem [[feedback-powershell-utf8]]
— luôn ghi lại bằng `[IO.File]::WriteAllText` + UTF8 no BOM vì file .cs đầy tiếng Việt.

**Lỗi CÓ SẴN phát hiện nhân tiện (không do refactor):** `M_SocialTest` fixture dùng id trần
`"small_talk"` trong khi `IdleActivityDef.ConfigErrors` đã có guard bắt thiếu tiền tố `core:` từ
2026-08-10 ⇒ suite đỏ 1 case suốt từ đó mà chưa ai chạy lại. Đã sửa thành `"core:small_talk"`.

**Kết quả:** biên dịch sạch 0 lỗi. Test: Decoupling 41/41 · WildCombat 53 · Health 96 · Mood 43 ·
Equipment 17 · EquipmentSlot 58 · Capability 8 · WorkAssigner 16 · M8 10 · Production 39 · Corpse 19 ·
Grief 28 · PawnAiBrain 4 · Needs 11 · NeedsSave 17 · Social 22 — **tất cả xanh**.
**ĐÃ VERIFY Play Mode 2026-08-15**: pack cắm đúng, ghi Realm đi thẳng vào sổ dịch vụ, `PawnBody` thuần
chạy đủ nhu cầu + `IsCultivator=false`, Hom phàm nhân 1 / Luyện Khí 3.50 / Đại Thừa 320.
(Pawn trong scene có `HomOverride=1` — là nerf pawn của M22, hành vi cũ có chủ đích, KHÔNG phải hỏng.)

**ĐÃ DỜI THƯ MỤC (cùng ngày, user yêu cầu tiếp):** `Traits`/`Skills`/`Capabilities` từ `Sim/Disciples/`
sang **`Core/Pawns/`**, namespace nay `Nizhangjie.Core.Pawns.{Traits,Skills,Capabilities}`.
⚠ **Dời PHẢI mang theo file `.meta` — của CẢ file lẫn THƯ MỤC** (`Traits.meta` nằm cạnh thư mục
`Traits/`), nếu không Unity cấp lại GUID. 25 file cập nhật namespace bằng 1 sweep `-creplace`, **biên
dịch sạch ngay lần đầu** (chứng tỏ không có tham chiếu tương đối kiểu `Traits.Trait` nào). Sau khi dời,
`Sim/Disciples/` chỉ còn đúng thứ thuộc về tu tiên/tông môn.

**ĐÃ DỜI TIẾP `ItemInstance`/`PlayerItemBag` → `Core/Items/`** (cùng ngày) — **layering nay SẠCH THẬT,
KHÔNG còn nợ**. Kèm `WeaponPassiveRoll` + `ItemQualityTier`. 46 file thêm `using Nizhangjie.Core.Items;`.

⚠ **Không dời nguyên si được, phải TÁCH ĐÔI:** 2 factory `ItemInstance.FromCraftResult`/`FromEquipmentId`
cần `CraftResult` + `EquipmentDatabase` + `WeaponPassiveRoller` — toàn thứ tầng Sim. Bê nguyên xuống Core
là chỉ ĐỔI CHỖ vi phạm chứ không gỡ. Giải: **dữ liệu món đồ ở `Core/Items/ItemInstance`, cách nó RA ĐỜI ở
`Sim/Crafting/ItemFactory`** (class MỚI). Ranh giới nhớ được: *"một món đồ không cần biết mình được rèn
ra, nhặt được, hay do quest thưởng."* 19 call site đổi `ItemInstance.From*` → `ItemFactory.From*`.

**Chốt tự động chống tái phát — `Arch_CoreNeverDependsOnSim`** (trong `PawnDecouplingTest`): quét TOÀN BỘ
`Scripts/Core/`, bắt **cả 2 dạng** — `using Nizhangjie.Sim.*` LẪN tên đầy đủ `Sim.X.` trong code (bỏ qua
dòng comment). ⚠ Bản ĐẦU của chính chốt này chỉ bắt `using` và **đã bỏ lọt** dạng tên đầy đủ — phát hiện
nhờ grep tay rồi mới siết lại; bài học: chốt kiến trúc phải kiểm cả 2 cách viết. Ngoại lệ CÓ CHỦ ĐÍCH duy
nhất: **`Core/Bootstrap/GameRoot.cs`** = composition root, được phép biết mọi tầng (luật §1).
**Dự án KHÔNG có asmdef nên layering thuần quy ước — test này là thứ DUY NHẤT giữ ranh giới**, vi phạm sẽ
không gây lỗi biên dịch nào.

Verify Play Mode (đợt dời item): `ItemFactory.FromEquipmentId("wep_iron_sabre")` ở Sim dựng ra
`Nizhangjie.Core.Items.ItemInstance` có roll passive thật (1 dòng), mặc vào pawn → ATK 46→72;
`PawnBody.ItemBag` là `Nizhangjie.Core.Items.PlayerItemBag`. Test lại: Decoupling 42 · Equipment 17 ·
EquipmentSlot 58 · Loot 30 · Passive 49 · WildCombat 53 · Production 39 · FixedMap 37 — tất cả xanh.

**Lỗi CÓ SẴN thứ 2 phát hiện lúc chạy hồi quy (KHÔNG do refactor):** `Run M6 Tests` đỏ 1 case —
`[M6 FAIL] QuestChain — stepUnlocked=False boardCount=0` (M6SimTest Test 8). Đã xác nhận vô can bằng
grep: `Sim/Quests/QuestChain*.cs` không hề tham chiếu Disciple/PawnBody/Trait/Skill/Capability. Đã tách
thành task riêng, chưa sửa.

Liên quan: [[project-m28-pawn-society]] (M28 làm tầng danh tính, M37 làm tầng thân xác — cùng một hướng),
[[feedback-core-sim-before-xianxia]] (nguyên tắc gốc "engine trước, xianxia là content-pack"),
[[project-m34-equipment-slots]] (cùng mô-típ: gỡ nội dung xianxia khỏi enum lõi ra JSON),
[[project-monster-design]], [[project-m29-companion]] (2 hệ từng phải fork vì khoá cứng này).
