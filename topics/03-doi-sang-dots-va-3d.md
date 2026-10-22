# Đổi sang DOTS (hybrid) rồi lên 3D: tôi để sẵn chỗ cho tọa độ thứ ba từ trước

**Thời gian:** 11/09 → 13/10/2026 · **Kỹ năng:** chuẩn bị kiến trúc cho thay đổi lớn, khảo sát bằng số, tách lớp hiển thị khỏi dữ liệu.

> Tóm gọn: tôi cố ý giữ mọi thứ **sẵn sàng cho chiều thứ ba** từ trước (save tách phẳng, tọa độ có thể mở rộng), nên khi nâng cấp, **chi phí thấp hơn nhiều** so với làm lại từ đầu. Kế hoạch dự kiến là "khoảng 2028–2030"; trên thực tế thế giới khối 3D chạy được từ **10/2026**.

## 1. Câu hỏi ngày 11/09
Tôi hỏi liệu có nên "nhân dày" hệ ô 2D thành voxel, để phòng khi dự án lớn thành game 3D thật (dự đoán 2028–2030), tránh làm vỡ API của modder và tránh phải viết lại trong tuyệt vọng. Ý tôi lúc đó rất nhẹ: chỉ cần 2 tầng Z (mặt đất / không phải mặt đất).

**Claude khảo sát bằng code thay vì đoán:**
- Toàn bộ **251 file, khoảng 1961 chỗ** dùng thẳng `Vector2Int`. Đổi sang `Vector3Int` là việc cơ học, trình biên dịch dẫn đường, nhưng *lớn thật*.
- **Tin tốt:** save/JSON đã an toàn từ trước, vì mọi snapshot dùng `int x, int y` phẳng, *không bao giờ* serialize `Vector2Int`. Thêm `int z = 0` sau này không làm hỏng save cũ. Đây là quy ước tôi đã áp dụng có chủ ý.
- Tạm thời hoãn đổi engine (chưa tính năng nào cần z), nhưng phần *schema save và mod*, thứ không sửa muộn được, đã an toàn sẵn.

## 2. Ngày 12/09: làm thật
Chỉ một ngày sau tôi quyết định làm. Quy trình an toàn:
1. `git init` và commit baseline **trước khi đụng file nào**.
2. Thử "bỏ qua hệ lớn, làm hệ nhỏ trước" — **thất bại và dừng lại**: đổi một hệ chiến đấu nhỏ gây **267 lỗi trải khắp ~80 file**. Kết luận từ số liệu: lưới ô là "mạch máu chung" của gần như toàn bộ gameplay, không có hệ nhỏ độc lập nào để nhặt. Việc phải đi từng lớp, có Play Mode kiểm tra giữa chừng, không phải "quất cho hết trong một đêm".
3. Chuyển theo lớp (lưới/vùng/tìm đường/chiến đấu/thế giới → tests/editor). Kết quả: **toàn project 0 lỗi** (3 commit). 242 file dùng `Vector3Int`; 97 file còn `Vector2Int` có chủ ý (thành phố, NPC cũ…), nối bằng ép kiểu ở biên. `z` luôn bằng 0, chưa tính năng nào dùng.

## 3. DOTS theo kiểu hybrid (01/10)
Tôi chốt dùng DOTS **kiểu lai**: chỉ Burst/Jobs cho những chỗ nóng, không chuyển cả game sang ECS. Bảy luật nằm trong `CLAUDE.md` của dự án. Lý do chính:
- ECS không vẽ được tilemap 2D; mod nạp lúc chạy không "bake" được; API Entities hay đổi sẽ phá mod C#. → **mod không bao giờ chạm kiểu ECS/NativeArray**.
- Dữ liệu nóng thuộc DOTS, lạnh thuộc Registry; điểm đồng bộ là `TickPhase`; ID số chỉ trong RAM, save vẫn lưu `namespace:id`; mỗi hệ DOTS giữ bản C# thường để test so sánh.

**Kết quả đã đo:**
- **A\* của NPC thành phố chạy Burst:** đường đi *giống hệt* bản C# từng ô, nhanh **×5–7** trên thành phố 400×400 (13 test).
- **Lõi noise địa hình tự viết, chạy Burst** trên luồng nền. Sau này hàm độ cao 3D cũng được giữ **giống bản C# từng bit** (luật C9).
- **Đo ngày 02/10: chưa hệ nào cần thêm DOTS.** Lỗi chậm ở Wild là do quét scene (đã sửa ×14), thành phố 1000 NPC p95 chỉ 9,4 ms. **Gói Entities chưa cài và chưa cần cài.**

## 4. Lên 3D thật (08–13/10)
Vì tọa độ đã là `Vector3Int` và save đã phẳng, bước lên thế giới khối chỉ là *mở rộng tầng thứ ba* chứ không phải xây lại:
- Hai bộ vẽ 2D (tilemap) và 3D (voxel) là **hai bản vẽ ngang hàng của cùng một dữ liệu**. Vị trí là dữ liệu (`CellPos`, `pos`), sim chỉ phát tín hiệu `Changed`, renderer tự nghe. Có test quét mã nguồn: ngoài renderer cấm gọi vẽ lại; lớp 3D cấm đọc sprite/transform.
- Tách assembly (`Nizhangjie`, `.Editor`, `.VoxelView`) để game không tham chiếu được lớp voxel.
- Sau đó 2D được "đóng kho", và thế giới trở thành khối 16³ sửa được ([case 07](../case-studies/07-the-gioi-khoi-render-anh-sang-nuoc.md)).

## Điều tôi rút ra
- **Chuẩn bị rẻ ở chỗ không sửa muộn được** (định dạng save, API mod) và **hoãn ở chỗ sửa muộn vẫn được** (đổi kiểu toàn engine).
- **Khảo sát bằng số trước khi quyết** (251 file, 267 lỗi) thay vì ước tính.
- **Không theo trào lưu:** chọn DOTS hybrid và *đo xong thấy chưa cần mở rộng* thì dừng.

## Bằng chứng
[`project_voxel_z_future_proofing.md`](../docs/lessons/project_voxel_z_future_proofing.md) · [`docs/VECTOR3INT_MIGRATION_NOTES.md`](../docs/VECTOR3INT_MIGRATION_NOTES.md) · [`project_dots_hybrid.md`](../docs/lessons/project_dots_hybrid.md) · [`project_render_2d3d_coupling.md`](../docs/lessons/project_render_2d3d_coupling.md) · [case 07](../case-studies/07-the-gioi-khoi-render-anh-sang-nuoc.md)
