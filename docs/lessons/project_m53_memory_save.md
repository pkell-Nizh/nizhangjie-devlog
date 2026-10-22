---
name: project-m53-memory-save
description: M53 — lưu sử ký + ký ức xuống save (trả nợ M48); bẫy nhân đôi ký ức khi Restore đi qua Record; ký ức KHÔNG suy lại được từ sử ký; đo kích thước save thật
metadata: 
  node_type: memory
  type: project
  originSessionId: 38f6f05f-a15c-443e-a952-6a5a0dacc2bf
  modified: 2026-08-21T14:37:08.791Z
---

**M53 (2026-08-21)** — trả nợ khai từ [[project_m48_chronicle]]. Trước đó load lại là **cả colony mất trí
nhớ**, đúng lớp bug tự mâu thuẫn mà M30 (permadeath) từng tránh cho HP.

## Kiến trúc

`SaveSystem/ChronicleIO.cs` + `MemoryIO.cs`, mirror `WorldEventIO` từng nét. Cắm ở **CẢ BA** đường
`SaveManager.Load`/`Create`/`WriteFile` — mọi lối lưu/nạp đều qua, không bắt từng nơi gọi tay rồi quên.

`MemorySaveEntry` phẳng hoá ký ức kèm chủ nhân vì **`JsonUtility` không serialize Dictionary** (bẫy đã
dính ở M40 với `WildQuestSeedDefinition`). Giữ `MemoryEntry` nguyên dạng struct lồng ⇒ thêm field về sau
không phải sửa 2 chỗ.

## ⚠⚠ BẪY NHÂN ĐÔI KÝ ỨC — lý do `WorldChronicle.ReplaceAll` tồn tại

Nạp sử ký mà lặp `Record()` thì mỗi bản ghi **fan-out xuống `MemoryRegistry` lần thứ hai**, và mỗi vòng
save/load là ký ức nhân đôi — im lặng, chỉ lộ sau vài vòng. `ReplaceAll` **cố ý KHÔNG bắn `Recorded`**.
Test `Restore_NoDoubleFanOut` khoá lại.

(Ngược lại `MemoryRegistry.ReplaceAll` **có** đi qua `Remember` — ở đó không có fan-out để nhân đôi, mà
lại được hưởng đúng luật lọc/ép trần hiện tại.)

## ⚠ KÝ ỨC KHÔNG SUY LẠI ĐƯỢC TỪ SỬ KÝ

Nghe thì tưởng được (sử ký có đủ sự việc, fan-out lại là xong) — **đó là cái bẫy**. Ký ức mang thứ sử ký
không hề biết: **độ tin đã tụt**, **nguồn tin**, **con số bị thổi phồng**, và cả mẩu mà chủ nhân **không
dự phần** (nghe đồn). Suy lại = biến mọi tin đồn thành sự thật tận mắt, xoá đúng phần thú vị nhất của M49.
Cùng lớp lỗi với `MoodRegistry.Restore` (M32): **ghi THẲNG trạng thái, đừng suy lại**.

## Quyết định khác

- **Trần HIỆN TẠI thắng trần lúc lưu** — mod hạ trần sau khi người chơi đã save thì sổ nạp vào theo trần
  mới (`ReplaceAll` gọi `PruneExpired` + `EnforceCaps`).
- **Save cũ → 2 sổ về rỗng**, không chặn load. Không cần sentinel: list rỗng vốn là trạng thái hợp lệ,
  khác `Hunger` phải dùng `-1f` vì 0 là giá trị thật (luật B4).
- **`Capture` khi chưa có sổ trong locator → GIỮ NGUYÊN save cũ**, không ghi đè bằng rỗng (mirror
  `WorldEventIO`) — chưa boot mà xoá sạch sử ký là mất dữ liệu không phục hồi.

## ⚠ Kích thước save — số đo thật

- Save bình thường (2 sự việc): **12.4 KB** cả file.
- Ép **xấu nhất** (sử ký đầy trần 2000 + 20 pawn đầy trần ký ức = 2530 bản ghi): **880.9 KB**.

Ca xấu nhất **thực chiến không tới**: trần THỜI GIAN (30 ngày × weight) cắn trước trần số lượng rất lâu —
2000 chỉ đạt được khi nhồi 2500 sự việc vào vài giờ in-game. Muốn siết thì hạ
`chronicle_config.json → maxEntries`, **không cần đụng code**.
`SaveManager` dùng `prettyPrint: true` nên file to gần gấp đôi bản nén — **cố ý không tự đổi** vì ảnh
hưởng toàn bộ save.

## Trạng thái

Test `Run Memory Save Tests` **34/34** — đi qua `JsonUtility.ToJson/FromJson` THẬT (lớp lỗi hay gặp nhất
của save không phải logic mà là "JsonUtility lặng lẽ không serialize được kiểu này").
Hồi quy: Chronicle 49/49, Memory 83/83, Exchange 42/42.

**ĐÃ VERIFY PLAY MODE 2026-08-21** — ghi/đọc **file save THẬT trên đĩa** qua `SaveManager.Save()`/`Load()`:
đánh nhau + trò chuyện thật → xoá sạch 2 sổ → nạp lại → sử ký đủ ô (101,100) + sentinel không-rõ-chỗ, ký
ức 3 chủ **không nhân đôi**, và **tin đồn giữ nguyên `tin=0.75` + nguồn + `value=33.2`** (khác số thật
31.65). Hội thoại sau khi nạp vẫn chạy.

⚠ Nhắc: bẫy MCP `execute_code` chặn `System.IO.File.Delete` — dọn file test phải bật `safety_checks=false`.

Liên quan: [[project_m48_chronicle]], [[project_m49_subjective_memory]], [[project_m52_dialogue_beats]].
