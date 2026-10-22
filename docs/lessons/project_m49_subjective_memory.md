---
name: project-m49-subjective-memory
description: "M49 — ký ức CHỦ QUAN từng pawn (tin đồn méo có tất định + diễn giải lại theo thiện cảm hiện tại) và hệ biến ký ức thành câu nói RNG, đã nối vào PawnDialoguePanel"
metadata: 
  node_type: memory
  type: project
  originSessionId: 38f6f05f-a15c-443e-a952-6a5a0dacc2bf
  modified: 2026-08-21T02:08:33.788Z
---

**M49 (2026-08-21)** — đợt 2/3, xây trên [[project_m48_chronicle]]. Hoàn tất đúng mục tiêu user chốt:
lời thoại **vừa RNG vừa phụ thuộc người nghe ⟂ người nói ⟂ diễn giải hiện tại**.

## Kiến trúc

`Core/Memory/`: `MemoryEntry` (struct) + `MemoryRegistry` + `MemoryOps` + `MemoryConfig`.
**Dùng CHUNG `MemoryEventDef` với sử ký** — không dựng bảng loại-sự-việc thứ hai.

**Đường vào tự động DUY NHẤT là sử ký** (luật B2): `MemoryRegistry.Install(chronicle)` cắm vào
`WorldChronicle.Recorded` (event của chính sổ, **không** qua `GlobalEventBus` — đây là "sổ nuôi sổ", không
phải thông báo 1-nhiều). Ngoại lệ có chủ đích: `Hear`/`TryGossip`.

**Ai được nhớ (bản này): người dự phần** (actor+target) — có mặt theo định nghĩa nên **không cần truy vấn
không gian**. Người ngoài đứng gần là bước sau (cần chỉ mục không gian; đừng quét toàn bộ pawn mỗi sự
việc — bài học C3 lag giờ đi làm). Tin vẫn lan nhờ `TryGossip` gắn vào `SocialInteractionOps.Apply`: **hai
người đã đứng nói chuyện rồi**.

## ⭐ Diễn giải lại — rẻ nhất mà đắt giá nhất

Bản ghi chỉ giữ sự việc THÔ; *ý nghĩa* tính LÚC ĐỌC từ thiện cảm **hiện tại** (`ToneOf` →
Warm/Neutral/Cold, có **vùng chết** bắt buộc luật B1). Không tốn thêm byte nào mà được: quan hệ xấu đi thì
mọi chuyện cũ tự nhuốm màu khác. Verify thật: cùng ký ức (tick 36, value 27.17 y nguyên), đổi opinion
−70→+70 thì câu đổi từ *"Anh đánh tôi mất 27.2 máu. Tôi nhớ rõ từng vết."* sang *"Tay anh nặng thật đấy.
Nhưng tôi không giận."*

## Quyết định đáng nhớ

- **Méo tin đồn dùng `DetHash` chứ KHÔNG `Random`** (luật A4) — độ tin tụt mỗi miệng, con số thổi phồng
  theo phần tin đã mất, dưới `minConfidenceToKeep` thì **tự tắt** (không lan vô hạn).
- **CỐ Ý KHÔNG méo danh tính** (đổi tên kẻ gây ra): nghe hấp dẫn nhưng biến mọi truy vấn "ai làm gì" thành
  rác, và người chơi đọc ra thành LỖI chứ không phải kịch tính. Để dành cho khi có cơ chế HIỆN được
  "người này tin sai".
- **Trần KHÁC sử ký có chủ ý**: sử ký cắt từ CŨ NHẤT (cuốn chiếu); ký ức bỏ mẩu **ĐIỂM THẤP NHẤT**
  (`weight × confidence`) — trí nhớ dài hạn phải giữ chuyện lớn dù cũ, và bỏ tin đồn mờ trước ký ức tận mắt.
- **`MemoryRole` SUY RA, không lưu** (Did/Suffered/Witnessed/Heard = f(tôi là ai, bản ghi)).
- **`MemorySpeechOps.Speak` gom MỌI cặp (ký ức × câu) rồi bốc 1 lần** — bốc 2 lần (chọn ký ức xong mới
  chọn câu) sẽ trúng ký ức chưa ai viết câu, rồi phải im hoặc bốc lại. **Sort pool trước khi bốc** vì
  `DataRegistry.GetAll()` là Dictionary (không sort thì cùng seed vẫn ra câu khác — bẫy A4).

## ⚠ Vì sao KHÔNG dùng lại `DialogueLineDef` (luật A6 bắt trả lời)

`DialogueLineDef` chọn **một CHÙM câu** cho panel VN theo bối cảnh TĨNH (opinion/quan hệ/chủng tộc), câu là
chuỗi cố định. `MemoryLineDef` chọn **một CÂU về một sự việc CỤ THỂ** theo 6 trục (loại sự việc, vai, sắc
thái, có dính người nghe không, độ tin, opinion với người nghe) và có SLOT. Nhồi 6 trục vào def cũ = mọi
file thoại cũ cõng thêm 6 field vô nghĩa + `DialogueOps.Pick` thành hàm 9 tham số. **Cái được tái dùng là
LUẬT CHỌN** (lọc-rồi-bốc-theo-trọng-số, không bao giờ "khớp đầu tiên"), không phải kiểu dữ liệu.

## ⚠ Bug nội dung bắt được lúc verify → đã thành guard vĩnh viễn

`{days}` nở ra **chữ thường** ("hôm nay"/"3 ngày trước"). Đặt đầu câu ra *"hôm nay anh còn động tay với
tôi."* — **không sai code nên không có gì báo**, chỉ đọc mới thấy sượng. Viết hoa tự động là sai hướng
(slot khác vốn là tên riêng), nên `MemoryLineDef.ConfigErrors` **chặn từ data**: bắt `{days}` ở đầu câu
hoặc ngay sau `.`/`!`/`?`. Đã sửa 6 file + verify guard bắt đúng bản cũ, bỏ qua bản mới.

## Đã nối vào game thật

`PawnDialoguePanel.Open` chèn câu-từ-ký-ức lên **ĐẦU** rồi mới tới chùm thoại chung (ký ức cụ thể phải đi
trước lời chào chép sẵn). Không có ký ức hợp ⇒ `HasLine=false`, chạy y như trước M49.

## Nợ đã biết

**Ký ức + sử ký đều CHƯA đi vào save** — load lại là quên sạch. Cần `SaveSystem/ChronicleIO` + `MemoryIO`
đợt sau (luật B5).

## Trạng thái

Test `Run Memory Tests` **83/83**. Hồi quy: Chronicle 49/49, Decoupling 42/42 (**arch guard xanh** —
`Core/Memory` sạch Sim), Social 22/22, Trade 84/84.
**ĐÃ VERIFY PLAY MODE 2026-08-21** (4 pawn thật, đường production): A đánh B (dmg 27.2) → A,B nhớ, C
không; đổi thiện cảm thì câu đổi mà ký ức y nguyên; tin đồn qua 2 miệng độ tin **1.00→0.75→0.50**, số
**27.17→28.16→29.99**; 6 seed ra 4 câu khác nhau, có seed bốc trúng **ký ức khác hẳn**; panel hội thoại
thật đổi câu đầu theo thiện cảm.

Liên quan: [[project_m48_chronicle]], [[project_dialogue_vn_panel]], [[project_m29b_social]],
[[project_m35_grief]], [[project_code_system_rule]].
