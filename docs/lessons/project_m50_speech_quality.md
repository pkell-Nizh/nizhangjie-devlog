---
name: project-m50-speech-quality
description: "M50 — chống lặp lời (33.1%→0.0% đo thật) + đại từ tiếng Việt theo quan hệ; hai đòn bẩy 'đủ tốt đủ nhỏ' nâng chất toàn bộ thoại cũ mà không viết thêm câu nào"
metadata: 
  node_type: memory
  type: project
  originSessionId: 38f6f05f-a15c-443e-a952-6a5a0dacc2bf
  modified: 2026-08-21T13:33:48.206Z
---

**M50 (2026-08-21)** — hai đòn bẩy chất lượng cho hệ thoại, chốt sau khi brainstorm "sinh thoại đủ tốt đủ
nhỏ". Đặc điểm chung: **không sinh thêm một chữ nội dung nào mà nâng chất toàn bộ nội dung đã có.**

## Cái bẫy đã đo được (lý do cả đợt tồn tại)

6 trục lọc của [[project_m49_subjective_memory]] = **192 ô**; muốn mỗi ô 3 câu là **~576 câu viết tay**.
⇒ **Luật rút ra: thêm một trục lọc là chuyện 5 phút trong code nhưng nhân đôi nợ nội dung vĩnh viễn.**
Từ giờ thêm trục phải trả lời được "ai sẽ viết 2× số câu".

## (1) Chống lặp — `Core/Social/SpeechHistory`

- Nhớ theo **CẶP (người nói → người nghe)**, không theo người nói: kể lại cho người MỚI là bình thường.
- Khoá = `defId#lineIndex` (**khung câu**), không phải chuỗi đã điền slot — người chơi nhận ra sự lặp ở
  khung, chứ "mất 12 máu"/"mất 15 máu" vẫn là cùng một câu.
- **HAI tầng, tầng 2 sinh ra TỪ SỐ ĐO chứ không từ suy đoán:**
  - phạt mềm → **33.1% → 10.5%** lặp liên tiếp (300 lượt × 10 câu, rổ 3 dòng). Vẫn nghe ra được.
  - **luật CỨNG** "cấm lặp NGAY câu vừa nói, MIỄN LÀ còn lựa chọn khác" → **0/2700 = 0.0%**, 0 lần im lặng.
  - Điều kiện "còn lựa chọn khác" giữ rổ-1-câu không làm pawn hoá câm — tệ hơn cả lặp lời.
- Rổ ứng viên đổi từ (ký ức × def) sang **(ký ức × def × DÒNG)** — chống lặp nhớ theo từng dòng, bốc def
  trước rồi mới bốc dòng thì hình phạt không có đường tác dụng.
- Tinh chỉnh nhét vào `MemoryConfig` — **không đẻ file config mới** cho 2 con số (mỗi file JSON mới là
  thêm một cặp phải giữ đồng bộ, luật C2).

## (2) ⭐ Đại từ theo quan hệ — `PronounOps` + `PronounDef`

**Đòn bẩy lớn nhất riêng của tiếng Việt.** Tiếng Anh có đúng một chữ "you" nên game phương Tây phải diễn
đạt quan hệ bằng NỘI DUNG câu. Tiếng Việt thì **cặp đại từ CHÍNH LÀ quan hệ** — một trục cõng thiện cảm +
tuổi + giới + thân sơ. Giải được nó thì mọi câu đã viết tự nhạy quan hệ, và trục `tone` bớt nhân đôi file.

- **BUG THẬT nó diệt:** `memory_lines` hardcode "anh/tôi", `dialogue_lines` hardcode "ngươi/ta" ⇒ **cùng
  một pawn nói "ngươi" ở câu chào rồi "anh" ở câu ký ức ngay sau**, trong một lượt. Nay panel giải MỘT lần
  rồi áp cho CẢ hai nguồn.
- **Cơ chế ở lõi, TỪ NGỮ ở content-pack** (luật A5) — kiếm hiệp "ta/ngươi", hiện đại "tôi/bạn", cùng cơ chế.
- **CHỌN TẤT ĐỊNH, không random** — cố ý khác chọn-lời-thoại: xưng hô nhảy qua lại là người chơi đọc ra
  LỖI. Dùng `priority`, hoà thì so `Id` (không để thứ tự Dictionary quyết định — bẫy A4).
- ⚠ **TỰ VIẾT HOA ĐẦU CÂU** — cùng lớp lỗi im lặng với `{days}` ở M49 nhưng **xử NGƯỢC LẠI**: `{days}` thì
  CẤM đặt đầu câu, đại từ thì KHÔNG cấm được (sẽ què nội dung) nên phải viết hoa. Viết hoa ở đây đúng
  tuyệt đối vì đại từ là danh từ chung — ngược hẳn `{other}`/`{listener}` vốn là tên riêng.
- ⚠ **`{other}` (TÊN) ⟂ `{you}` (ĐẠI TỪ)** — gọi thẳng tên người đối diện trong tiếng Việt nghe rất khách
  sáo, nên câu nói với chính đương sự phải dùng `{you}`.
- **Chốt chống tái phát:** test quét MỌI câu thoại thật và làm đỏ nếu còn "ngươi/mày/tao" hardcode.

## Bar "đủ tốt" đã chốt (test được)

> Nói chuyện với cùng một pawn **10 lần liên tiếp**: không câu nào lặp, và ít nhất 3 lần nhắc tới sự việc
> có thật trong ký ức.

Vế đầu **ĐÃ ĐẠT** (0/2700). Vế sau còn phụ thuộc độ phủ nội dung — rổ hợp bối cảnh hiện chỉ ~3 dòng.

## Còn lại trong lộ trình (chưa làm)

3. **Mảnh ghép có tag sắc thái** (mở đầu × lõi × đuôi, 16 mảnh → 144 câu mặt) — đòn bẩy KHỐI LƯỢNG. Làm
   sau (1)(2) vì nó NHÂN LÊN chất lượng skeleton. Nguy cơ: ghép sai tiểu từ cuối câu ("đấy/nhỉ/à/thôi") lộ
   giọng máy ngay ⇒ ràng buộc ghép-cùng-tag phải nằm trong `ConfigErrors`.
4. Nhiều nhịp trong một lượt. 5. Nhân bản offline bằng LLM (**không** LLM lúc chạy — phá determinism).

**Nếu tới (3) mà thấy 144 câu mặt đọc dở hơn 16 câu viết tay chỉn chu thì DỪNG ở (2)** — đó là kết cục hợp
lệ, không phải thất bại.

## Trạng thái

Test `Run Speech Tests` **44/44**. Hồi quy: Memory 83/83, Chronicle 49/49, Trade 84/84, Decoupling 42/42.
**ĐÃ VERIFY PLAY MODE 2026-08-21** — cùng cặp pawn thật, chỉ đổi thiện cảm, cả lượt đổi giọng nhất quán:
thù → "mày/tao", trung tính → "ngươi/ta", thân → "anh/tôi".

Liên quan: [[project_m49_subjective_memory]], [[project_m48_chronicle]], [[project_dialogue_vn_panel]].
