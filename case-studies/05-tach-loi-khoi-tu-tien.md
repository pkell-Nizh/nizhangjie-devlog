# Case study 05 — Tách "lõi mô phỏng" khỏi "nội dung tu tiên" (M37) và biến nó thành luật có test

**Thời gian:** 15/08/2026 (M37), được củng cố 28–30/09 · **Kỹ năng:** refactor kiến trúc lớn an toàn, đo quy mô trước khi làm, giữ tương thích ngược.

## Vấn đề
Dự án bắt đầu như game tu tiên, nhưng mục tiêu thực của người làm là **một engine mô phỏng thế giới dùng được cho mọi loại nội dung** (tu tiên chỉ là một "content pack"). Thực tế code lại khoá cứng vào lớp `Disciple` (đệ tử tu tiên).

## Khảo sát bằng số liệu trước khi làm
- `Disciple` xuất hiện ở **199 file**; `new Disciple(...)` ở **35 file**; tầng trừu tượng danh tính mới (`PawnRef`) mới chỉ ở 28 file.
- Ba mắt xích khoá cứng: `WildPlayerToken.Unit` alias thẳng tới `Disciple`; `PawnAiContext.Pawn` là `Disciple` ⇒ **cả 17 JobGiver** khoá theo; `Hunger/Rest/Mood/HP` là field *trên* `Disciple` ⇒ thú và NPC vĩnh viễn không biết đói/mệt.
- Bằng chứng của cái giá: hệ Companion và Beast đã phải **fork hẳn đường ống AI riêng** để né `Disciple`.

## Thiết kế — ba mảnh
1. `PawnBody` — thân xác sống tổng quát. **`Disciple` trở thành lớp con** ⇒ 199 file cũ biên dịch nguyên vẹn, không phải sửa (refactor *bằng kế thừa ngược* để tránh big-bang).
2. `CultivationRegistry` + `CultivationProfile` — sổ tu vi tách riêng, khoá theo `PawnRef`. `ProfileOf()` trả `null` cho phàm nhân và đó là **hợp lệ**. `Disciple.Realm/QiLayer` chỉ còn là *proxy*.
3. `XianxiaContentPack.Install()` — **điểm cắm duy nhất**.

> **Luật một câu:** hệ thống *mới* nhận `PawnBody`, không nhận `Disciple`. Cần tu vi thì *hỏi* registry, đừng *đòi* kiểu.

### Phép thử sống-hay-chết
Bỏ `XianxiaContentPack.Install()` thì game vẫn chạy đủ (đói/mệt/suy sụp/làm việc/đánh nhau/chết), chỉ là mọi người có hệ số `Hom = 1`. Một test khoá điều khoản này: **ngày nào nó đỏ = có người kéo tu tiên ngược vào lõi.**

## Quyết định cẩn trọng
- `PawnRef` **giữ tiền tố `"disciple:"`** vì đó là *định dạng dây* đã nằm trong save + combat — đổi là vỡ mọi save.
- Chế tác/trang bị/job là *cơ chế lõi*, còn *nội dung* mới là tu tiên → nới engine, giữ JSON.
- `Loyalty` ⟂ `Mood` cố ý không cộng dồn (tập thể vs cá nhân).
- UI tự ẩn dòng "Cảnh Giới/Linh Căn" với phàm nhân.

## Bẫy đã dính — và bài học
Sweep đổi tên kiểu bằng PowerShell `-replace` (**mặc định không phân biệt hoa thường**) nuốt luôn `(disciple == null` thành `(PawnBody == null` (5 chỗ, chỉ lộ lúc biên dịch). `\bDisciple\b` còn khớp 3 lớp ngoài *kiểu*: segment namespace, tên thành viên, field của event — mỗi lớp im lặng cho tới compile. → **Sweep đổi tên phải dùng `-creplace`**, ghi thành bẫy C4 trong bộ luật. Và: file `.cs` đầy tiếng Việt phải ghi UTF-8 *không BOM*.

Một lỗi có sẵn cũng lộ ra: một fixture test dùng id trần `"small_talk"` trong khi guard bắt thiếu tiền tố `core:` đã có từ 10/08 → suite đỏ 1 ca suốt từ đó mà chưa ai chạy lại.

## Kết quả
Biên dịch 0 lỗi; test **Decoupling 41/41**, WildCombat 53, Health 96, Mood 43, Equipment 17, EquipmentSlot 58, Capability 8, WorkAssigner 16, Production 39, Corpse 19, Grief 28, Needs 11, NeedsSave 17, Social 22 — tất cả xanh. Verify trong Play Mode: ghi `Realm` đi thẳng vào sổ dịch vụ; `PawnBody` thuần chạy đủ nhu cầu; hệ số `Hom` phàm nhân 1 / Luyện Khí 3,50 / Đại Thừa 320.

## Hậu quả sau này (đã chứng minh lõi tách thật)
Cuối tháng 9 (xem [case study 11](11-mod-va-content-pack.md)) toàn bộ nội dung tu tiên và Tensura được **cất thành mod tắt được** (`xianxia`, `tensura`), còn Pokémon thành mod `pokemon` — game lõi vẫn chạy.

## Bằng chứng
[`docs/lessons/project_m37_core_xianxia_split.md`](../docs/lessons/project_m37_core_xianxia_split.md) · luật A5 trong [`docs/CODE_SYSTEM_RULE.md`](../docs/CODE_SYSTEM_RULE.md)
