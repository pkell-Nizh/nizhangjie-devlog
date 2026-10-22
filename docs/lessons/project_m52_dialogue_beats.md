---
name: project-m52-dialogue-beats
description: "M52 — lượt thoại 3 nhịp (chào → ký ức → khép lại); lộ ra 2 bug thật: lines ngược nghĩa giữa 2 bảng def, và 3 nhịp dùng chung 1 dòng chống lặp"
metadata: 
  node_type: memory
  type: project
  originSessionId: 38f6f05f-a15c-443e-a952-6a5a0dacc2bf
  modified: 2026-08-21T14:19:51.264Z
---

**M52 (2026-08-21)** — bước (4) lộ trình thoại, xây trên [[project_m51_speech_fragments]].

Trước M52 một lượt thoại là **danh sách câu phẳng** lắp ngay trong `PawnDialoguePanel.Open`: không test
được, và "thứ tự các câu" chỉ tồn tại dưới dạng thứ tự mấy dòng `_lines.Add(...)`.

## Kiến trúc

`Core/Social/DialogueBeat.cs` + `DialogueExchangeOps.Build()` — kéo việc lắp xuống Core, thứ tự thành
**hợp đồng có tên** khoá được bằng test.

**3 nhịp:** `Greeting` → `Memory` (M49) → `Reaction` (dẫn vào cột lệnh tương tác).
Nhịp thiếu nội dung thì **vắng mặt**, không chèn câu độn.

`DialogueLineDef.beat` rỗng = `Greeting` ⇒ mọi file thoại cũ chạy y nguyên. `Memory` không khai được ở
bảng đó (`ConfigErrors` chặn).

## ⚠ ĐẢO NGƯỢC quyết định M49 có chủ ý

M49 đặt nhịp ký ức LÊN ĐẦU ("cụ thể phải đi trước lời chào chép sẵn"). **Lý lẽ đó sai khi đã có cấu trúc
nhịp**: người ta CHÀO trước rồi mới vào chuyện — mở màn bằng *"mày đánh tao hôm nay"* mà chưa chào câu nào
nghe như cắt giữa băng. Cái M49 thật sự muốn tránh là *lời chào nhạt rồi hết*, giờ nhịp 2+3 lo phần đó.
Test `Order_MemoryIsNotFirstAnymore` khoá lại để không ai "sửa" về như cũ.

## ⚠⚠ HAI BUG THẬT lộ ra khi verify (đều đã thành test)

**1. `DialogueLineDef.lines` ⟂ `MemoryLineDef.lines` NGƯỢC NGHĨA NHAU.**
Bản đầu đổ hết mọi câu của chùm thành nhịp riêng ⇒ lượt xã giao phình thành **5 lần bấm**. Đào ra mới thấy
`DialogueLineDef` được ghi là "dãy tuần tự" còn `MemoryLineDef` là "biến thể" — **cùng tên field, nghĩa
ngược**. M52 thống nhất về BIẾN THỂ (bốc 1). Nội dung cũ nhiều câu vẫn chạy, hoá ra ĐA DẠNG HƠN trước.
- Nhân tiện lộ luôn: câu thứ 2 của mỗi chùm chào cũ (*"Nói nhanh đi, tao không có hứng"*) thật ra đang làm
  việc của nhịp PHẢN ỨNG — hồi đó chưa có nhịp 3 nên bị nhét chung. Đã tách đúng chỗ.

**2. Cả 3 nhịp dùng CHUNG một dòng lịch sử chống lặp.**
"Câu vừa nói" mà nhịp KHÉP đem so lại chính là câu KÝ ỨC vừa ghi ngay trước đó ⇒ luật cứng so nhầm đối
tượng, mất tác dụng. Đo được: nhịp khép lặp liền kề **1/4 lượt**. Sửa: `SpeechHistory` thêm **kênh**
(`channel`), mỗi nhịp một dòng riêng. Kênh mặc định rỗng ⇒ chỗ gọi cũ không đổi hành vi.

## Mở rộng kèm theo

- `DialogueOps.PickLine` (mới) bốc tới tận (chùm × DÒNG) — cùng lý lẽ `MemorySpeechOps`. `Pick` cũ giữ
  nguyên chữ ký, nay là vỏ gọi `PickLine`.
- **Chùm CHÀO/PHẢN ỨNG nay cũng chống lặp** (M50 mở rộng) — trước chỉ câu-từ-ký-ức được bảo vệ, nên bắt
  chuyện 5 lần là nghe đúng một lời chào 5 lần (nhàm nhất vì nó là câu ĐẦU mỗi lượt).
- Mảnh ghép (M51) + đại từ (M50) áp cho MỌI nhịp qua `DialogueExchangeOps.RenderPlain`.

## Trạng thái

Test `Run Exchange Tests` **42/42**. Hồi quy: Fragment 33/33, Speech 44/44, Memory 83/83, Trade 84/84.
**ĐÃ VERIFY PLAY MODE 2026-08-21** — panel thật, 5 lần bắt chuyện liên tiếp: đúng **3 nhịp** mỗi lượt,
**cả ba nhịp lặp liên tiếp 0** (chào 3/5 dạng · ký ức 5/5 · khép 3/5).

Còn lại: (5) nhân bản offline bằng LLM (**không** LLM lúc chạy).
⚠ Nợ treo từ M48: **sử ký + ký ức CHƯA vào save**.

Liên quan: [[project_m51_speech_fragments]], [[project_m50_speech_quality]],
[[project_m49_subjective_memory]], [[project_dialogue_vn_panel]].
