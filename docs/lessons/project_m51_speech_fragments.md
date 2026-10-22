---
name: project-m51-speech-fragments
description: "M51 — ghép mảnh thành câu (đòn bẩy khối lượng); bar 'đủ tốt' đạt cả hai vế 10/10 khác nhau; bug thiết kế tone-rỗng bị che do test bắt được"
metadata: 
  node_type: memory
  type: project
  originSessionId: 38f6f05f-a15c-443e-a952-6a5a0dacc2bf
  modified: 2026-08-21T13:57:57.701Z
---

**M51 (2026-08-21)** — bước (3) lộ trình thoại, xây trên [[project_m50_speech_quality]]. M50 giải "đừng
lặp" + "xưng hô đúng"; bước này giải **"lấy đâu ra đủ câu"** (bài toán 192 ô × 3 câu ≈ 576 câu viết tay).

## Phân vai — lý do KHÔNG nhét mảnh vào `MemoryLineDef`

Skeleton giữ **lõi đặc thù từng loại sự việc** và đã lo toàn bộ lọc bối cảnh. Mảnh giữ **phần biến thiên
DÙNG CHUNG** (mở đầu, đuôi) tái dùng được qua mọi loại sự việc. **Mảnh không thuộc về loại sự việc nào
cả** — đó là toàn bộ lý lẽ tách def.

`SpeechFragmentDef` (`Data/speech_fragments/`, 16 file): `group`/`tone`/`text`/`weight`.
Skeleton gọi bằng `#nhóm#`.

## ⚠ THỨ TỰ XỬ LÝ BẮT BUỘC

**ghép mảnh → điền slot dữ liệu → điền đại từ (M50) → `Tidy`**

Ghép TRƯỚC để mảnh chứa được slot; đảo lại thì `{you}` nằm trong mảnh lộ nguyên ra màn hình.

`Tidy` **không phải mỹ phẩm**: mảnh RỖNG là biến thể hợp lệ và phổ biến nhất, luôn để lại khoảng trắng
đôi / dấu phẩy mồ côi / chữ đầu câu viết thường. Không dọn thì thấy `" , anh đánh tôi"` — y hệt lỗi hiển thị.

## ⚠⚠ BUG THIẾT KẾ THẬT do test bắt được

Bản đầu: "mảnh ĐÚNG sắc thái thắng tuyệt đối, mảnh `tone` rỗng chỉ làm dự bị."
Hậu quả: `frag_open_none` (**mảnh RỖNG, trọng số cao — biến thể quan trọng nhất**) **không bao giờ được
bốc** khi nhóm có mảnh Cold/Warm ⇒ **mọi câu đều bị đeo một cái mở đầu**, đúng cái *"nghe rất máy móc"* mà
chính doc của nó cảnh báo. Im lặng, không lỗi, chỉ nghe mới ra.

**Sửa:** `tone` rỗng nghĩa là **HỢP MỌI SẮC THÁI** ⇒ vào rổ CÙNG mảnh đúng sắc thái, không phải dự bị.

## Lưới an toàn

- `SpeechFragmentOps.ValidateAllLoaded()` (GameRoot mục 6g) bắt câu gọi `#nhóm#` không tồn tại. Phải ở
  tầng này vì `ConfigErrors` từng def chỉ thấy chính nó — mirror `EquipmentDefinition.ValidateAllLoaded()`.
  Không bắt thì token bị bỏ IM LẶNG: câu vẫn đọc trôi nên **không ai biết một nửa biến thể đã chết** (C1).
- Nhóm lạ lúc chạy → bỏ token sạch sẽ, **không để lộ `#...#`** (thấy dấu thăng là thấy lỗi; thiếu mở đầu
  thì không ai nhận ra).
- Cấm ghép LỒNG (`#` trong `text`) — đường thẳng tới vòng lặp vô tận.

## CỐ Ý CHƯA làm

**Mảnh cỡ TIỂU TỪ** ("đấy/nhỉ/à/thôi/cơ") — biên giới rủi ro nhất: tiểu từ sai sắc thái phá câu nặng hơn
dùng sai từ, và cần mô hình ngữ pháp chứ không phải bảng tra. Mảnh hiện tại đều là **mệnh đề đứng được một
mình** — ghép sai thì cùng lắm rời rạc chứ không sai ngữ pháp.

## ⚠ Bẫy test

`DataRegistry.LoadFromFolder` **CỘNG DỒN, không xoá** (đã đọc nguồn để chắc). Test đăng ký def giả mà còn
để `_loaded=false` thì mỗi lần `Expand` sẽ đổ data THẬT chồng lên ⇒ đo nhầm nội dung thật. Cách ly đúng:
gọi `EnsureLoaded()` MỘT lần cho cờ bật, RỒI mới `Clear()` + đăng ký def giả.

## ⭐ BAR "ĐỦ TỐT" ĐÃ ĐẠT CẢ HAI VẾ

> 10 lần liên tiếp với cùng một pawn: không câu nào lặp, và nhắc tới sự việc có thật.

Diễn tiến: M49 chỉ **3** dạng → M51 lần đầu **6/10** → sau khi chỉnh nội dung **10/10, lặp liên tiếp 0**
(đo ở cả 2 sắc thái, Play Mode, pawn thật, cùng một ký ức).

**Hai lần chỉnh là NỘI DUNG chứ không phải code**: hạ trọng số mảnh RỖNG 3→2 (60% lượt không mở đầu là quá
lấn át) + cho MỌI skeleton dùng đủ 2 khe (trước chỉ 2/3 dòng).

## Trạng thái

Test `Run Fragment Tests` **33/33** (§3 đo thẳng phép nhân: 3×4 = đúng 12 câu mặt từ 7 chuỗi).
Hồi quy: Speech 44/44, Memory 83/83, Chronicle 49/49.
Còn lại: (4) nhiều nhịp trong một lượt · (5) nhân bản offline bằng LLM (**không** LLM lúc chạy).

Liên quan: [[project_m50_speech_quality]], [[project_m49_subjective_memory]], [[project_m48_chronicle]].
