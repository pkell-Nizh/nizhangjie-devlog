---
name: project-m54-speech-audit
description: "M54 — trình soát nội dung thoại + nhân bản lấp lỗ; 4 ô thủng→0, câu mặt 70→564 (×8.5); công cụ tự có 2 điểm mù đã sửa"
metadata: 
  node_type: memory
  type: project
  originSessionId: 38f6f05f-a15c-443e-a952-6a5a0dacc2bf
  modified: 2026-08-21T15:09:36.966Z
---

**M54 (2026-08-21)** — bước (5), bước CUỐI của lộ trình thoại (M48→M54).

## Diễn giải lại đề bài

"Nhân bản offline bằng LLM" — rủi ro thật KHÔNG phải *"viết được chữ hay không"* mà là **300 câu tầm
thường/hỏng lặng lẽ lọt vào game**. Nên việc đáng làm ở bước này là **cái đo**, chữ chỉ là hệ quả.
Thứ tự: **soát trước → sinh sau → soát lại** (đúng tinh thần luật C3 "đo trước, sửa sau").

## Công cụ: `UI/Editor/SpeechContentAuditor.cs`

Menu `Nizhangjie → Audit Speech Content`. Bắt **ba kiểu hỏng đều IM LẶNG**:
- câu trùng/gần trùng (rổ nhìn to mà nghe vẫn nhàm),
- ô bối cảnh RỖNG (pawn im bặt, không lỗi gì),
- xưng hô hardcode (phá hệ đại từ M50).

Kèm độ phủ (chào/khép × 5 dải thiện cảm; ký ức × loại × vai × sắc thái) và **khối lượng thật**
(chuỗi viết tay → câu MẶT sau ghép mảnh).

`Report(out int problems)` trả CHUỖI, `Run()` chỉ là vỏ log — vì **console MCP không đọc lại được log
nhiều dòng** (bẫy gặp ngay lúc làm).

## ⚠⚠ Công cụ tự có 2 ĐIỂM MÙ — soi kỹ báo cáo đầu tiên thay vì tin ngay

1. **Thăm dò bằng độ tin 1.0** ⇒ mọi câu vai `Heard` (khai `maxConfidence` thấp) trượt hết ⇒ **báo thủng
   oan**. Sửa: độ tin thăm dò phải HỢP VAI (`Heard` → 0.5).
2. **Báo thủng cho tổ hợp KHÔNG BAO GIỜ tới được** — sự việc không có đối tượng (`{actor} qua đời`) thì
   vĩnh viễn chẳng ai mang vai `Suffered`. Sửa: suy từ `MemoryEventDef.RequiresTarget`, không hardcode.

**Giới hạn thành thật đã ghi vào tool:** tổ hợp trống hoàn toàn thì liệt ra mục riêng và **KHÔNG tự kết
luận là lỗi** — công cụ không đoán được ý đồ tác giả. Còn đúng 1 tổ hợp trống có chủ ý:
`mem_killed × Suffered` (nạn nhân chết rồi, không kể lại được).

## Phát hiện quan trọng nhất

**4 ô thủng đều ở Neutral** — tức pawn im bặt ở đúng dải quan hệ **phổ biến nhất** (−20..+20). Nặng hơn
hẳn cảm giác ban đầu, và **chỉ thấy được nhờ có công cụ đo**. Kèm theo: `mem_attacked × Did` ("chính tôi
ra tay") **trước giờ không hề có câu nào**.

## Kết quả

| | trước | sau |
|---|---|---|
| ô thủng | 4 | **0** |
| chuỗi viết tay | 58 | 86 |
| **câu MẶT** | 70 | **564** |
| hệ số nhân | ×1.7 | **×8.5** |

Test hồi quy: Memory 83/83, Fragment 33/33, Speech 44/44, Exchange 42/42. Auditor báo **SẠCH**.
**ĐÃ VERIFY PLAY MODE 2026-08-21** — dải trung tính nay nói được, 6 lượt ra 5/6 câu khác nhau.

## Chốt về LLM (giữ nguyên qua cả lộ trình)

Nội dung do LLM viết **lúc soạn data**, xuống JSON tĩnh — runtime vẫn thuần lọc-rồi-bốc, tất định, mod sửa
được, chạy offline. **KHÔNG dùng LLM lúc chạy.** LLM giải bài toán SỐ LƯỢNG, không giải hợp-ngữ-cảnh —
chính auditor mới là thứ giữ cho số lượng khỏi biến thành rác.

Liên quan: [[project_m53_memory_save]], [[project_m52_dialogue_beats]], [[project_m51_speech_fragments]],
[[project_m50_speech_quality]].
