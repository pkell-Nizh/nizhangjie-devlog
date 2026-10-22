---
name: project-m28-pawn-society
description: "M28 Universal Pawn System — doc thiết kế chốt 3 tier danh tính, PawnRef thay vì gộp class, Role là gói không phải cổng runtime"
metadata: 
  node_type: memory
  type: project
  originSessionId: cdf61f38-9f79-4df7-9cfa-24a8eb2c6dbd
  modified: 2026-08-07T17:05:20.171Z
---

2026-08-06: user + Gemini brainstorm ra "Universal Pawn System" (Skyrim/RimWorld, "không có NPC giả").
Tớ viết `Assets/Scripts/Nizhangjie/M28_PAWN_SYSTEM_DESIGN.md` chốt kiến trúc.
**B1 (danh tính bền) XONG cùng ngày — test 62/62 xanh + ĐÃ VERIFY PLAY MODE** (doc §14.8: bấm Hồ Sơ 3
NPC → "Rời Thành" → vào lại → `Nhập hồn 3/3 Actor vào 'settle_1'`). Còn đường B (save round-trip) chưa
verify tay. B2-B7 chưa làm; §13 còn 6 câu hỏi user chưa chốt (câu 5 chặn B2).

**2 lỗi TỚ tự gây trong B1, cùng một gốc — quên mất "player chạm được chưa":**
① Viết bước verify "thoát ra world map rồi vào lại" mà KHÔNG kiểm có đường ra không — City chưa hề có
nút về WorldMap (Wild có `WildHUDController._backButton`, City không). Đã thêm
`World/UI/ReturnToWorldMapButton.cs` + nút "Rời Thành" wire sẵn vào `CityScene` qua MCP.
② Mọi nhánh thất bại của `CityNpcSystem.Notice()` đều `return null` CÂM (+ `?.` nuốt thêm 1 nhánh) ⇒
user verify không ra log thì không chẩn được gì. **Lặp lại đúng bài học ForestLifecycle/DayNight đã ghi
sẵn trong CLAUDE.md.** Luật rút ra: code hạ tầng mới thì nhánh THẤT BẠI phải kêu to hơn nhánh thành
công, và phải log CẢ khi kết quả = 0 (phân biệt "rỗng" ⟂ "có nhưng không khớp"). Xem
[[feedback-done-means-playable]].

**Ý niệm cốt lõi của cài đặt B1 — LINH HỒN ⟂ THÂN XÁC** (đáng nhớ vì nó là lời giải cho mâu thuẫn #1):
`NpcRecord` = thân xác (tái sinh từ seed mỗi lần vào thành, ephemeral); `Core.Identity.ActorRecord` =
linh hồn (vào save, trần 300). Vào thành → `NpcActorOps.HydrateInto` nhập hồn vào một thân xác **CÓ
SẴN** (không cấy thêm người ⇒ dân số không phình mỗi lần load). `NpcRecord.ActorRef` rỗng = dân nền
thuần. Người có ActorRef **miễn nhiễm turnover** (không thì họ biến mất giữa phiên ngay trước mắt
player). Chỉ lưu 6 core stat, KHÔNG lưu `CharacterStats` nguyên khối (combat stat là dẫn xuất — lưu
lại là mời modifier sống lại chồng nhau mỗi lần load).

**Vì sao doc này tồn tại:** plan gốc của Gemini không biết dự án đã có sẵn 60% (M27 schedule,
`IsOutdoors` lazy-eval, capability system, `Sim/Quests/StorytellerSystem`, `Sim/Economy`) và mâu thuẫn
với 2 quyết định đã chốt trong code.

**3 mâu thuẫn + cách gỡ (đây là phần đáng nhớ, không nằm trong code):**
1. `NpcRecord` **ephemeral, không vào save** ⟂ plan đòi thừa kế/trả thù nhiều năm → gỡ bằng **3 tier
   danh tính** (Actor persistent ~300 / Resident ephemeral 1000 / Abstract số gộp) + **promote một
   chiều** khi NPC "được chú ý". Nông dân thành tông chủ = promote lúc nhặt sách, không phải sim 1000
   người 5 năm.
2. Plan đòi gộp `Disciple`+`NpcRecord` thành class `Pawn` → **KHÔNG**. Thống nhất ở **địa chỉ**
   (`PawnRef` string prefix, quy ước đã có từ combat `unitId`) + `IPawnIdentity` + **side-table**
   (memory/social/role là Dictionary rời, KHÔNG field trên class) — side-table chính là thứ cho mod
   thêm trục dữ liệu mới mà không fork `Disciple.cs`.
3. `PawnRole` suýt thành trục phân loại thứ 3 song song capability+Tier → chốt Role = **gói khai báo
   lúc SINH** (mirror `PawnKindDef`), **cấm xuất hiện trong `if` của JobGiver/JobDriver**. Ngoại lệ duy
   nhất: tầng xã hội/giá/hội thoại. Xem [[feedback-verb-strings-english]] cùng họ luật "1 trục thôi".

**Bác bỏ có lý do (§8):** ECS-lite/đa luồng của plan — chi phí thật đã đo là A* 317ms chứ không phải
cache locality, và đa luồng trái luật determinism. "Dictionary tra O(1) tiết kiệm bộ nhớ" là sai tiền
đề — thứ giữ chi phí bé là TRẦN số Actor.

**§12–13 bổ sung cùng ngày sau 2 vòng phản biện với user — 3 chốt quan trọng hơn phần trên:**

1. **BẾ QUAN là nút bấm của player, không phải edge case.** Tớ xếp thừa kế/vòng đời vào "chỉ thấy khi
   chơi 50h+" → **SAI, đo nhầm thước**: thể loại tu tiên đo bằng NĂM in-game, player lên Nguyên Anh giờ
   thứ 8 rồi bấm cho 50 năm trôi. Không có thừa kế thì xuất quan xong cả thành là nhà hoang. ⇒ tách
   HAI thao tác thời gian: *vắng mặt* (catch-up trần 30 ngày) ⟂ *bế quan* (`GenerationalPass` duyệt
   theo năm, KHÔNG trần, **bắt buộc có bảng báo cáo XUẤT QUAN** đọc từ Chronicle). Bảng đó là thứ
   biến mô phỏng thừa kế từ vô hình thành cơ chế.
   → **Luật chung cho cả dự án:** trước khi kết luận "player có thấy hệ này không", phải hỏi *"thời
   gian in-game trôi nhanh cỡ nào"*. Có nút tua 50 năm ⇒ mọi hệ theo tuổi/thế hệ là hệ TUYẾN ĐẦU.
2. **Chuỗi cung ứng: giữ nhưng ở mức GỘP theo thành/làng, KHÔNG theo từng NPC** (chốt của user). Đúng
   cả 3 trục: player chỉ tương tác với *cái chợ* · mức gộp đã có sẵn (`CityState`+`MarketDynamics`) ·
   và là mức DUY NHẤT sống sót qua bế quan 50 năm. Cắt tầng cá thể = chọn đúng độ phân giải, không
   phải bỏ tính năng.
3. **Bark (thoại phản ứng) là khoảng trống Skyrim lớn nhất** — engine ~200 dòng, nhưng cần ~900 dòng
   thoại. User bác lập luận "quá nặng content" và đúng: sinh bằng LLM **ở khâu authoring** (offline →
   JSON → người duyệt → commit; **KHÔNG runtime**, sẽ phá determinism+save) + bản thân bark là bề mặt
   modding tốt nhất. Rủi ro thật không phải số lượng mà là **độ chung chung** → chữa bằng token thật
   (`{kin_name}`, `{place_name}`) lấy từ Chronicle + casting ⇒ bark/Chronicle/casting phải làm CÙNG NHAU.

Cũng chốt: tầng **Faction làm nền + cá nhân là lớp đè** (`FactionDefinition.cs` đã có sẵn mà bản đầu bỏ
quên) — gỡ được hạn chế "giết người vô danh thì không ai phản ứng"; cờ **`Essential`** chống hệ emergent
tự ăn mất nội dung (bài học Oblivion); **sở hữu+nhân chứng+tội phạm nâng khỏi nợ** vì không có luật thì
không có gì để "emergent" chống lại.

**§13 ĐÃ CHỐT HẾT 2026-08-06 (doc §15):**
- **Ngân sách Actor tách ĐÔI: 500 giữ-vĩnh-viễn + 1000 quen-mặt** (ĐÃ CODE, test 74/74). Lý do tách
  chứ không phải một số lớn: tông môn địch 50 người có tên không được để 50 ông bán hàng lỡ bấm xem
  đẩy ra. Trần cũ 300 nêu vì "save phình" là SAI — đo lại 1000 actor ≈ 800KB, 5000 ≈ 4MB.
- **Ngưỡng promote giữ RỘNG** (mở hồ sơ = vào sổ). An toàn nhờ sửa bug: `TryReclaim` bản đầu chỉ đuổi
  người CHẾT ⇒ ai cũng sống ⇒ chạm trần là kẹt vĩnh viễn. Nay quen-mặt bị quên được **kể cả còn sống**
  (LRU), người có hệ quả thì không. + **nâng hạng** Noticed→Traded/Harmed (không thì người vừa giao
  dịch xong vẫn nằm diện bị quên). + **`AddPinProvider` hook có TỪ B1** để B4 cắm social/memory vào —
  chính rủi ro user nêu ("làm loạn cây relationship nếu NPC despawn hẳn").
- **Tội phạm tách đôi**: làm sở-hữu+nhân-chứng (cấp nguyên liệu cho bark), hoãn lính-canh+nhà-tù.
- **Lộ trình đổi**: faction + bark TRƯỚC social graph cá nhân. Mức nhắm B2 chỉ **80 dòng thoại**
  (20 điều kiện × 4), không phải 900 — con số cũ làm nó đáng sợ quá mức.

**Kiến trúc thoại = bản của USER, tốt hơn "bảng tra" tớ đề xuất** (doc §15.5): lai 2 tầng —
**Grammar Nodes + weight-RNG** cho nhân vật thường, **Fixed Diagram ít node viết tay** cho nhân vật có
cốt truyện. Cắt tải theo LOD: trong tầm → sinh câu thật; ngoài tầm → **chỉ ghi Social Log**, tương tác
vẫn xảy ra nhưng không được diễn đạt. **Social Log ⟂ Chronicle hoá ra là MỘT thứ** (nguồn cắt tải +
token cho bark + bảng XUẤT QUAN) — hợp nhất, đừng dựng hai.
⚠ **Bẫy đã chốt trước khi code**: dev tiếng Việt → sau chuyển tiếng Anh, nhưng **grammar RULE phải
tách theo NGÔN NGỮ**, chỉ dùng chung hợp đồng slot — không dịch lá cây được (trật tự từ khác, và
**xưng hô** ta/ngươi/lão phu/tiền bối/tiểu tử đổi theo tuổi+vai vế+quan hệ mà tiếng Anh chỉ có "you").
Đây là CƠ HỘI: NPC đổi "tiểu tử"→"tiền bối" sau khi player đột phá = cơ chế thật, rất đúng chất tu tiên.

**B2a FactionMatrix ĐÃ CODE 2026-08-06** (doc §16, test 46/46, chưa verify Play Mode). Phát hiện:
`Data/factions/*.json` + `FactionDefinition` đã nạp vào GameDatabase từ lâu mà **người tiêu thụ duy
nhất là cửa sổ Odin** — data nằm chờ hệ. 4 quyết định: **lưu LỆCH khỏi nền JSON** (cân bằng lại data
vẫn ăn vào ván cũ) · **đối xứng** (một chiều chỉ có nghĩa ở tầng cá nhân) · **id lạ = trung lập CỐ Ý**
(mỗi thành tự là một thế lực nhỏ, không phải author 100 file — đây là xô hứng "cả thành ghét ngươi",
gỡ hạn chế §2.5) · **kẹp phần LỆCH chứ không chỉ kẹp lúc đọc** (không thì lệch trôi vô cực, "làm việc
tốt bù lại" thành bất khả thi). B2a mới là hạ tầng — CHƯA ai gọi `Modify` từ sự kiện thật.

**2 bẫy mới, đáng nhớ cho cả dự án:**
- **Đừng dán ký tự vô hình vào source.** Dán thẳng US (0x1F) làm dấu phân cách khoá thì code *chạy
  đúng* nhưng editor hiện như 2 nháy rỗng và mọi công cụ xử lý sai (diff/grep/khớp-chuỗi-để-sửa). Dùng
  escape `''`.
- **Test "tự bỏ qua khi thiếu data" là test GIẢ.** Test nạp JSON của B2a ban đầu chỉ chạy nếu database
  đã sẵn — trong Editor thì chưa → 39/39 xanh mà mối nối đáng kiểm nhất chưa từng chạy. Sửa: test tự
  `DataRegistry.LoadFromFolder(...)`. Luật: test phải TỰ dựng tiền đề, không được skip im lặng.

**Grammar = BỘ LUẬT theo ngôn ngữ, cắm rời** (user chốt: sẽ cần zh/ja/ko/en/vi, "tiếng Việt là còn
ít"); **tiếng Anh làm bản tham chiếu TRƯỚC** vì hình thái đơn giản nhất → lõi sạch, rồi **xưng hô là
phần MỞ RỘNG** cắm vào thay vì nướng đặc thù tiếng Việt vào lõi. Chỉ dùng chung *hợp đồng slot*.
Insight: tiền bối = 前辈 = 先輩 senpai = 선배 sunbae — **xưng hô generalise qua cả CJK+VI, chỉ tiếng Anh
là ngoại lệ** ⇒ xây 1 lần dùng 4 ngôn ngữ.

**B2b CHIÊU MỘ ĐÃ CODE 2026-08-06** (doc §17, test 49/49, chưa verify Play Mode) — **chèn TRƯỚC Social
Log vì phát hiện nút thắt: `CityNpcProvider` chỉ có ĐÚNG 1 verb "Profile" và nó chỉ mở bảng đọc** ⇒ B1
và B2a đều là hạ tầng KHÔNG có đường chạm, xây tiếp là thành 4 tầng chồng nhau mà player chưa làm được
gì với NPC nào.
- **KHÔNG dùng `Recruitment.Recruit()`**: nó đòi `Sect`, mà **CityScene không có `LocalMapManager` ⇒
  không có Sect**. Mượn công thức giá `Recruitment.ComputeCost` nhưng giao dịch chạy ở tầng City qua
  `CityJobBootstrap.Player.Wallet`; người mới vào thẳng `DiscipleRegistry.Get()` toàn cục ⇒ vòng lặp
  chạm được: chiêu mộ → thấy ở Character Menu → mang sang Wild làm pawn.
- **Chuyển hạng THỨ TƯ `actor:` → `disciple:`** (bản đầu chỉ tính 3 hạng): bản ghi Actor **KHÔNG bị xoá
  mà thành BIA MỘ** (`becameDiscipleId`/`HasMoved`/`RedirectedRef`) — xoá đi là để lại con trỏ treo cho
  B4 mà không lỗi nào bắn ra. Chiêu mộ cũng nâng hạng sang `Recruited` ⇒ vào ngân sách giữ-vĩnh-viễn.
- Nút nằm **trong panel hồ sơ** (user chốt: quyết định sau khi đọc stats, chừa ô verb context menu cho
  trò chuyện/khiêu chiến/nhận nhiệm vụ/trao đổi). Nút LUÔN hiện, chỉ mờ + LUÔN kèm lý do.
- **Faction TRUNG TÍNH** (user chốt) ⇒ B2b cố ý KHÔNG gọi `FactionMatrix.Modify`. Ép hệ quả vào chỗ
  không thuộc về nó chỉ để "dùng cho có" thì tệ hơn để trống.

**Sau B2b: chốt HAI ĐỒNG TIỀN tách biệt (doc §18)** — user hỏi tưởng nhỏ "chiêu mộ trừ ví nào" lộ ra bug
đặt tên có TRƯỚC cả M28. Chốt: **Linh Thạch** = tài nguyên tu luyện phân cấp, LOCAL, gắn `Sect`
(`SectResources`) — bản chất không sinh ra để giao dịch. **Bạc** = tiền giao dịch thật, GLOBAL, sống
trong `PlayerWallet`/`PlayerProfile`, không cần `Sect` (đây là lý do nó tồn tại — hành động liên-map
như chiêu mộ cần ví không phụ thuộc có tông môn hay không).
**Phát hiện quan trọng: KHÔNG cần đổi cấu trúc gì cả** — `PlayerWallet` từ trước đã đúng y hệt thứ
user mô tả (global, sống xuyên scene qua save, không phụ thuộc Sect). Vấn đề CHỈ là nhãn: mọi UI hiện
số dư ví này đều gọi nhầm là "linh thạch" — kể cả `TimeControlBar` đã sai từ TRƯỚC M28, chỉ không ai
để ý vì chưa có SectResources.LingThach đứng cạnh để so sánh. Sửa 7 file, thuần đổi chữ, không đổi
hành vi runtime, test không cần sửa gì (không kiểm chuỗi hiển thị). Hệ quả hay: 2 đường chiêu mộ giờ
tiêu 2 đồng khác nhau và điều đó ĐÚNG (City không có Sect ⇒ bắt buộc Bạc; đường gốc trong Sect thật
dùng Linh Thạch tông môn).
→ **Bài học chung:** khi sửa 1 tính năng nhỏ mà thấy tên gọi/nhãn không khớp đúng cấu trúc code, hỏi
ngay — có thể là bug đặt tên nằm sẵn từ trước, chỉ chưa ai chạm tới để lộ ra.

**NGÂN HÀNG (Bank) ĐÃ CODE 2026-08-06** (doc §19, test 44/44, chưa verify Play Mode) — user hỏi
"đổi nguyên thạch ra bạc thì túi nào" mở ra hệ kinh tế mới, tách khỏi Pawn System gốc.
3 phát hiện trước khi thiết kế: **`Sect` KHÔNG chạy trong game thật** (grep GUID LocalMapManager trong
mọi scene = 0 kết quả — chỉ sống trong test) nên "đổi nguyên thạch" phải là đổi MATERIAL Wild chứ
không phải SectResources; **`MaterialDefinition.value` đã nằm sẵn trên mọi material, chưa ai đọc**
(cùng mẫu FactionDefinition trước B2a); **`CityBuildingProvider` cố ý CHƯA gắn verb nào cho công
trình nào** — Bank là tương tác công trình City ĐẦU TIÊN có hành vi thật.
Ranh giới kiến trúc quan trọng: Ví (`PlayerWallet`) chỉ sống ở City, Material (`Disciple.ItemBag`)
sống toàn cục qua `DiscipleRegistry` nhưng `WildStorageInstance` (kho theo địa điểm) KHÔNG — chỉ giữ
1 LocalMap active. ⇒ **Bank đổi từ TÚI ĐỆ TỬ, không đổi từ kho** — không phải thiếu tính năng mà là
giới hạn thật của kiến trúc lưu trữ.
MVP nhẹ theo user chốt: vay/trả/lãi kép theo ngày (neo GameClock, idempotent/ngày), KHÔNG hậu quả
cưỡng chế (tránh hình phạt vô hình). Tái dùng `WildBuildMaterialMenu` popup có sẵn thay vì dựng grid
mới (đúng luật M17.2 "float menu cho chọn đơn giản").
**Bẫy đáng nhớ:** kiểm ký tự font phải duyệt CẢ `characterLookupTable` chính LẪN
`fallbackFontAssetTable[].characterLookupTable` — suýt kết luận nhầm font "thiếu dấu tiếng Việt" (và
nghi oan cả nút Chiêu Mộ đã dựng trước đó) vì chỉ kiểm atlas chính (Static, chỉ có Latin-1), trong khi
dấu tiếng Việt thật nằm ở fallback asset riêng mà TMP tự resolve lúc render.

Liên quan: [[project-city-npc]] · [[project-city-npc-brain]] · [[project-m27-schedule]] ·
[[project-data-registry]] · [[project-time-pacing]] · [[feedback-done-means-playable]]
