# Case study 13 — Độ tin cậy: lưu trữ công trình, Unity treo 11 GB RAM, và đo RAM bản build

> **Tóm lược 3 dòng**
> - **Nhầm mục đích:** Tưởng map "trông như cũ" nghĩa là dữ liệu còn nguyên; lưu game ghi bản cũ.
> - **Thiết kế đúng ra sao:** Lưu snapshot và khôi phục trước khi dựng pathfinder; chụp ô đang sống trước khi ghi.
> - **Bài học:** Trạng thái mới phải được hỏi: "có sống qua save/load không?"; ghi rõ cái chưa phủ.
>
> *Các con số bên dưới lấy từ ghi chú đo lúc đó, không phải thứ cần thuộc lòng.*

## A. Công trình biến mất khi quay lại ô (M61, 28/08/2026)
**Lỗ hổng:** `WildSceneBootstrap.Generate()` tái sinh `LocalMapData` hoàn toàn mới mỗi lần vào lại một ô → mọi building/tường/kho/trạm người chơi dựng *biến mất im lặng*. Địa hình tất định theo seed nên *trông như map cũ*; chỉ nhà là mất thật. Tôi hỏi thẳng "cái gì là lỗ hổng cơ" rồi chốt vá.

**Thiết kế:** snapshot POCO phẳng (6 loại: BuildPiece, WildBuilding, Storage kèm *nội dung túi*, Workstation kèm *toàn bộ WorkOrder*, Producer kèm lịch, GroundItem vật liệu), dùng *chung công thức khoá ô* với hai sổ đã có (vị trí pawn, giường) — "không đẻ hệ song song". Footprint nén thành *hình chữ nhật origin + w×h* (xác minh bằng đọc code rằng mọi lớp dựng footprint kiểu `for x for y`).

**Điểm mấu chốt về thứ tự:** `Restore` phải chạy *trước* khi dựng pathfinder — thử qua sự kiện `OnGenerated` trước thì **sai** (event bắn *sau* khi pathfinder dựng) → nhà khôi phục không chặn đúng ô, pawn đi xuyên tường vừa hiện lại. Cố ý *không* gọi `ActivateAll` khi restore (sẽ dựng bản song song rỗng đè lên).

**Phạm vi nói thẳng:** *chưa phủ* — vùng/đánh dấu việc (đổi liên tục), tài nguyên đã khai thác (reset), trang bị rơi xuống đất.

**Bug tự tìm ra khi nối save:** "bấm Lưu mà *không rời map trước*" sẽ lưu bản **cũ hoặc rỗng** của chính ô đang đứng (snapshot chỉ chụp lúc rời map). Sửa bằng `CaptureCurrentSceneIfLoaded()` gọi đầu `WriteFile`. Ghi rõ: cùng lớp lỗi ở vị trí pawn (tầng 1) **chưa vá**, để khỏi quên.

**Kiểm chứng:** `Run BuildPersist Tests` — **31 ca**, round-trip riêng từng loại + qua `JsonUtility` thật.

## B. Unity treo, RAM leo 9–11 GB (29/08/2026)
- **Triệu chứng:** mở dự án kẹt ở splash "Initializing InitializeOnLoad Types", RAM leo tới 9–11 GB, không mở xong.
- **Nguyên nhân chính:** cache `Library/` hỏng do bị kill cứng nhiều lần giữa lúc import (còn các file `*-lock`). Xoá `Library/` để dựng lại → hết.
- **Đồng phạm phụ tìm ra khi điều tra:** một class `[InitializeOnLoad]` tự sinh asset mỗi lần nạp assembly; vì class `BleedDefinition` khai *bên trong* file khác tên, Unity không tạo `MonoScript` → asset ghi `m_Script: {fileID: 0}` (hỏng) → lần sau load null → guard lọt → ghi đè lại → import → reload → **vòng lặp**. Vi phạm luật JSON-first. Gỡ `[InitializeOnLoad]`, xoá 4 asset hỏng (đã grep: 0 code đọc chúng).
- **Bẫy khi sửa:** `Set-Content -Encoding utf8` (PowerShell 5.1) ghi kèm BOM → Hub báo manifest.json "not valid JSON, Char: 65279". Phải ghi UTF-8 *không* BOM.
- Một lỗi đỏ còn lại (TMP `Cull` NRE trong Unity 6) được ghi là *bug Editor đã biết, vô hại*, không đụng.

## C. RAM bản build (đo 21/10/2026)
Build Development + Profiler, lấy mẫu `Get-Process` mỗi 5 s: màn hình chính 0,6 GB WS → vào thế giới 2,6 GB WS (5–6 GB private) → sau 2 phút đi 3,3 GB WS / 7,2 GB private. Tôi treo game 5 giờ thật (~1 tháng 18 ngày trong game): RAM gần như không tăng ⇒ **không rò rỉ theo thời gian mô phỏng**. Chưa thử đi xa liên tục qua nhiều ô. *Chưa phân rã theo nhóm* — ghi là bước tiếp theo (nối Memory Profiler), không giả vờ đã giải xong. Bẫy: `manage_build` qua MCP làm mất kết nối → dùng `BuildPipeline.BuildPlayer` trong `delayCall` rồi chờ file kết quả.

## Bằng chứng
[`project_m61_tang3_build_persist.md`](../docs/lessons/project_m61_tang3_build_persist.md) · [`project_editor_open_hang_2026-08-29.md`](../docs/lessons/project_editor_open_hang_2026-08-29.md) · [`project_build_ram_2026-10-21.md`](../docs/lessons/project_build_ram_2026-10-21.md)
