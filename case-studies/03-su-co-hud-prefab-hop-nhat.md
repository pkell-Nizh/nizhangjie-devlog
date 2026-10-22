# Case study 03 — Sự cố prefab dùng chung và quyết định "hợp nhất có chủ đích"

**Thời gian:** 27/08/2026 · **Kỹ năng:** xử lý sự cố, phân tích kiến trúc, phục hồi khi không có bản sao lưu.

## Sự cố
`HUD.prefab` là **một prefab dùng chung** cho hai scene (Wild và City). Người dùng bấm *Apply All* ở City rồi *Revert* ở Wild: Apply đẩy toàn bộ khác biệt của City vào prefab gốc, Revert kéo Wild về khớp đúng file đó → Wild mất sạch phần riêng của nó.

**Không có đường phục hồi tự động:** lúc đó dự án *chưa dùng git*; không OneDrive, không File History, không Shadow Copy (đã kiểm tra hết); Undo của Editor cũng cạn vì chuyển scene xoá stack.
> Sau sự cố này dự án chuyển sang dùng git (lịch sử commit bắt đầu ngày 12/09).

## Quyết định thiết kế (phần đáng đọc nhất)
Thay vì cố dựng lại *y hệt* trạng thái phân kỳ cũ, người dùng chốt: Wild và City bản chất là **một**, chỉ khác bộ nạp dữ liệu — "giống chunk trong Minecraft" — và **giả định cũ "pawn không sống ở City" đã hết hạn**. → sửa **City** cho bắt kịp, không dựng lại Wild cũ.

### Nguyên tắc rút ra — *không phải thứ gì cũng nên hợp nhất*
Trong code có một comment ghi rằng dự án đã từng thử dùng chung `WildCursorReadout` cho City rồi **phải tách ra** (kiểu `WildTilemapRenderer` ≠ `CityTilemapRenderer`, field trống lặng câm). Câu hỏi cần hỏi trước khi hợp nhất bất cứ thứ gì hỏng:

> *"Khác biệt này có lý do kỹ thuật thật, hay chỉ là tình cờ chưa ai dọn?"*

Câu trả lời quyết định sửa ở prefab gốc (hợp nhất) hay sửa override từng instance (giữ tách):
- `HealthAlertBar`/`HealthPanel` → vào **prefab gốc** (lý do vắng mặt ở City đã hết hạn).
- `CursorReadout` → sửa **chỉ ở Wild** (khác biệt có chủ ý, loại renderer thật sự khác).

## Cách phục hồi (kỹ thuật)
- Mở prefab qua **Prefab Stage** (đúng đường sửa prefab gốc), không lặp lại lỗi "sửa instance rồi Apply".
- Giữ nguyên class `WildHealthPanel` 398 dòng, chỉ dựng lại hierarchy rồi nối ~20 field qua `SerializedObject`; dùng đúng hằng số cột đọc từ `WildHealthRow.cs`, không đoán.
- Trước khi gắn hệ vào City, đọc code xác nhận an toàn (ví dụ `TickBeasts` có guard thoát sớm khi không có `BeastSpawner`).

## Bẫy công cụ ghi lại
- `get_hierarchy` trả 783 object / 332 KB — vượt giới hạn → đọc qua file và lọc bằng PowerShell.
- `find_gameobjects` bỏ sót object có tên kèm khoảng trắng cuối (`"CursorReadout "`) → thao tác thẳng bằng instanceID.

## Bằng chứng
[`docs/lessons/project_hud_prefab_unify.md`](../docs/lessons/project_hud_prefab_unify.md)
