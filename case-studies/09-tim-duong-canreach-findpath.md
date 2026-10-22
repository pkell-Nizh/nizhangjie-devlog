# Case study 09 — "Tới được" nhưng "không tìm được đường": truy lỗi tìm đường 3D (17/10/2026)

**Kỹ năng:** gỡ lỗi hệ thống tìm đường phân tầng (region → corridor → A*), thiết kế phép đo công bằng, giả thuyết bị bác bỏ.

## Triệu chứng
`CanReach(a, b) == true` (kiểm tra nhanh trên đồ thị region) nhưng `FindPath(a, b) == null` (A* cấp ô). Đo trên WildLocal, 400 cặp quanh điểm vào, seed 12345: **87/252 cặp "tới được" bị lệch**.

## Giả thuyết 1: "hệ region nói dối" → **bị bác bỏ**
`PathRegionGrid` flood/link dùng đúng `CanStepSpots` như A*. Region đúng.

## Nguyên nhân thật (hai loại)
1. **80 cặp:** đường thật rất vòng (ví dụ thẳng 42 ô, đường 2431 bước quanh đồi) → A* hành lang 3D chạm trần số nút (30.000). Mục cũ trong hàng đợi vẫn được "mở" và ăn trần.
2. **7 cặp (+4 sau sửa lộ ra):** ô có `Cell.Occupant` là *công trình* — A* coi là chắn (trừ đích) còn region bỏ qua occupancy → lệch. Bỏ qua occupancy thì 11/11 có đường. → **câu hỏi thiết kế còn mở** (không phải bug thuần): công trình nên chặn trong PathGrid hay không. Chúng tôi *không tự quyết*, ghi lại.

## Sửa (commit 63e34077)
Heuristic = max(octile, **khoảng cách đồ thị region tới region đích**); bỏ qua nút đã đóng; phá hoà theo `h`; trần = max(30000, 2 × số chỗ đứng hành lang). Đường octile/Burst giữ thứ tự cũ. **Lệch 87 → 11 (toàn Occupant).** Thêm test `LongDetour_*` — *đỏ trên code cũ* (chứng minh test thật sự bắt được bug).

## Điều đã thử và bỏ
Heuristic "nhắm tâm region kế tiếp" → **tệ hơn trên núi** (tâm region lồi lõm nằm sau vách). Ghi lại để không ai thử lại.

## Bẫy phép đo (suýt đưa ra số giả)
Đo khi map chưa nạp xong (`Regions.IsPendingAt` còn true) cho số đẹp giả (87 → 7). Quy tắc: **chờ pending = 0, và so cũ ⟂ mới trong cùng một phiên Play.**

## Chuyện tiếp theo: làm tìm đường "không chặn" (17/10)
Một báo cáo "đi bộ vẫn khựng" dẫn tới chuỗi sửa:
- **Dân thành giật giờ đi làm:** gốc là `Regions.Version` tăng ~2 lần/giây (bất kỳ ô nào trong 5×5 đổi) → kết quả đường nền bị vứt + cache bị xoá → tính lại corridor mãi (270–380 ms/chuyến). Sửa: kiểm từng ô đường (`PathStillWalkable`) thay vì so version; corridor search *chạy tiếp được* với ngân sách 2 ms/khung. **DrainPathQueue tệ nhất 274 → 11,7 ms; khung ≥ 8 ms 131 → 20.**
- **"Tới được" dựng dần:** BFS hai chiều chạy tiếp được; `CanReach` đồng bộ chỉ còn cho chuyến ngoại ô. Tệ nhất 6,4 ms.
- **Quái không chặn luồng chính:** A* cấp ô chạy trên ThreadPool, chỉ giao yêu cầu mới nhất; quái giữ đường cũ khi chờ. `Beast.PathStep` max 2,1 ms (trước 20–35).
- **Dân trong nhà** (commit 77ca34fb): 25 → 7 ms.
- **Claude đo lại để khỏi phỏng đoán:** 22 chỗ gọi `CanReach` đồng bộ trong các `JobGiver_*`: đo thật (150 s, 4 pawn) cho **0 lần loang BFS** → *không cần* đổi hợp đồng JobGiver (một việc lớn chúng tôi định làm). Chỉ chuyến ngoại ô của dân thành đắt (max 11,9 ms).
- Một chi tiết trung thực: con số 6,7 ms của `Beast.PathStep` trong Profiler hoá ra **do Profiler thổi phồng**; log tạm cho thấy 110 s chỉ 6 lần vượt ngân sách (0,6–1,6 ms).

## Bằng chứng
[canreach_findpath_gap](../docs/lessons/project_canreach_findpath_gap.md) · [walk_hitch](../docs/lessons/project_walk_hitch_2026-10-17.md) · test `Run RegionPathfinder Tests`
