# Đổi sang DOTS (hybrid) rồi lên 3D: tôi để sẵn chỗ cho tọa độ thứ ba từ trước

**Thời gian:** 11/09 → 13/10/2026 · **Kỹ năng:** chuẩn bị kiến trúc cho thay đổi lớn, khảo sát bằng số, migration an toàn, đưa mã sang Burst mà giữ kết quả y hệt.

> Tóm gọn: tôi cố ý giữ mọi thứ **sẵn sàng cho chiều thứ ba** từ trước (save tách phẳng, tọa độ có thể mở rộng), nên khi nâng cấp, **chi phí thấp hơn nhiều** so với làm lại từ đầu. Dự kiến ban đầu là "khoảng 2028–2030"; trên thực tế thế giới khối 3D chạy được từ **10/2026**.

## 1. Câu hỏi ngày 11/09
Tôi hỏi liệu có nên "nhân dày" hệ ô 2D thành voxel, để phòng khi dự án lớn thành game 3D thật (dự đoán 2028–2030), tránh làm vỡ API của modder và tránh phải viết lại trong tuyệt vọng. Ý tôi lúc đó rất nhẹ: chỉ cần 2 tầng Z (mặt đất / không phải mặt đất).

**Claude khảo sát bằng code thay vì đoán:**
- **251 file, ~1961 chỗ** dùng thẳng `Vector2Int` (kiểu có sẵn của Unity, không có struct tự viết để chêm `z`). Đổi sang `Vector3Int` là việc cơ học, trình biên dịch dẫn đường, nhưng *lớn thật*.
- **Tin tốt:** save/JSON đã an toàn từ trước, vì mọi snapshot dùng `int x, int y` phẳng, *không bao giờ* serialize `Vector2Int` (có comment giải thích trong code). Thêm `int z = 0` sau này không làm hỏng save cũ.
- Quyết định: phần *schema save và mod*, thứ không sửa muộn được, đã an toàn sẵn → **hoãn đổi engine** cho tới khi tôi chủ động giao việc. Đó là hoãn có chủ ý, không phải quên.

## 2. Ngày 12/09: làm thật, và nó không suôn sẻ
Tôi giao việc theo từng khối nhỏ và nhiều lần dừng lại. Nhật ký migration ghi rõ từng bước:
1. **An toàn trước:** `git init` và commit baseline *trước khi đụng file nào*.
2. **Lần thử "làm hệ nhỏ trước" thất bại:** đổi lưới chiến đấu (tưởng độc lập) làm lỗi biên dịch nhảy lên **267 lỗi trải ~80 file**, kéo cả tìm đường, gần hết `World/Wild`, và sinh thành phố. Kết luận từ số đo: lưới ô là **"mạch máu chung"** của gần như toàn bộ gameplay, không có "hệ nhỏ độc lập" nào để nhặt. Dừng lại, không "quất cho hết" khi không ai theo dõi.
3. **Đi từng lớp, kiểm tra sau mỗi lớp bằng console của Unity:**
   - Tìm đường lõi (7 file): sạch trong phạm vi, nhưng tổng lỗi lại tăng 267 → 312 vì lộ ra tầng phụ thuộc phía sau. Ghi rõ đó *không phải* hồi quy.
   - Chiến đấu (27 file): sạch 100%.
   - `World/Wild`: tôi giao "quất cho hết trong đêm, không cần giám sát"; dọn ~280 lỗi về 0. Bài học: các struct "mạch máu ẩn" như `PawnAiContext` (gói tham số cho cả 17 JobGiver) tự biên dịch được nhưng vỡ khi cascade tới, nên sweep tự động không đủ.
   - Ranh giới cố ý giữ `Vector2Int`: `ITileWorldMapper`, `CaptureOps`, một phần thành phố/NPC; nối bằng ép kiểu ở biên.
4. **Một lỗ hổng quy trình được ghi lại thẳng:** các lần "verify sạch" ở ba phiên đầu chỉ kiểm assembly runtime, **chưa bao giờ kiểm assembly của test**. Khi kiểm, lộ thêm 500+ lỗi (bị giới hạn 500) trong các bài test; phiên đó cố ý dừng đúng phạm vi nhiệm vụ.
5. **Một sự cố nhỏ:** sửa một file bằng PowerShell làm hỏng mã hoá tiếng Việt toàn bộ comment trong file (mojibake); phục hồi bằng công cụ ghi lại đúng UTF-8, và ghi thành luật "file có tiếng Việt chỉ sửa bằng Edit".

Kết quả (commit `b8010d4` baseline → `92f3188` → `f3e98fc`): **toàn project 0 lỗi**; về sau 242 file dùng `Vector3Int`, 97 file còn `Vector2Int` có chủ ý, `z` luôn bằng 0 cho tới khi voxel xuất hiện.

## 3. DOTS theo kiểu hybrid (01/10)
Trước đó tôi giữ luật "mô phỏng đơn luồng" vì sợ race condition. Tôi cho rằng hệ thống an toàn của Jobs đã xử lý được chuyện đó, nên chốt dùng DOTS **kiểu lai**: chỉ Burst/Jobs cho chỗ nóng, không chuyển cả game sang ECS. Bảy luật nằm trong tài liệu dự án; những luật chính:
- **Sở hữu theo field:** dữ liệu nóng (vị trí, đường đi) thuộc DOTS, dữ liệu lạnh (tên, quan hệ, ký ức) thuộc Registry.
- **Điểm đồng bộ là `TickPhase`;** sự kiện từ job đi qua hàng đợi, hook mod chỉ phản ứng *sau* sự việc.
- **ID số chỉ trong RAM**, save luôn lưu `namespace:id`.
- **Mỗi hệ DOTS giữ bản C# thường** kèm test so hai bản cho kết quả giống hệt.
- **Mod không bao giờ chạm kiểu ECS/NativeArray** (Entities không vẽ được tilemap 2D, mod nạp lúc chạy không bake được, API Entities hay đổi sẽ phá mod C#). **Gói Entities chỉ cài nếu đo cho thấy cần.**

## 4. Hai bẫy khi đưa mã sang Burst (và cách tôi chứng minh nó đúng)
Mục tiêu: A* của NPC thành phố chạy Burst mà **đường ra giống hệt bản C# từng ô**. Ban đầu hai bản ra đường *khác nhau* dù logic chép y hệt:
1. **Mono không làm tròn kết quả trung gian về float32.** Đo: một biểu thức lệch bản chuẩn 2,4% số lần; viết liền hay tách qua biến tạm lệch nhau 20%. Lệch ở bit cuối là đủ đổi thứ tự phá hoà → **26/310 truy vấn ra đường khác**. Sửa: ép `(float)(...)` từng bước ở cả hai bản (đo lại 0/2.000.000 lệch).
2. **`SortedSet.Add` bỏ khoá trùng, heap thì không.** Port cấu trúc dữ liệu phải port cả *ngữ nghĩa ngầm*. Bug này chỉ lộ trên **thành phố thật** (~1/300 truy vấn); bản đồ ngẫu nhiên nhỏ không bắt được.

Luật tôi rút ra: test đối chiếu phải chạy trên dữ liệu thật cỡ thật, và phải được chứng minh là **đỏ khi bỏ phần sửa**, vì "test chưa từng đỏ thì chưa chứng minh được gì". Kết quả: Burst nhanh **×5,4–7** trên thành phố 400×400, 13 test.

**Điều không port được thì nói thẳng:** `Mathf.PerlinNoise` chạy trong Burst ra kết quả khác bản engine (~50% điểm lệch, tới 0,049). Nên tôi tự viết noise riêng làm tròn tường minh; kết quả: sinh một ô thế giới **92 → 19 ms**. Cái giá: **thế giới thay đổi** (núi nhiều hơn khoảng 10 điểm phần trăm, rừng ít hơn), nên tôi cho xem so sánh trước và chưa chỉnh tham số. Một chi tiết ngược trực giác: làm tròn tường minh khiến bản C# chậm hẳn ×5 ở truy vấn lẻ, nên vòng lặp lớn phải đi bản Burst theo vùng (một test từ 69 s thành 261 s rồi 48 s sau khi sửa).

## 5. Đo xong thấy chưa cần mở rộng (02/10)
Đo Profiler 8× trong Play Mode: **chưa hệ nào cần thêm DOTS.** Wild chậm ~5 ms/tick *không phải* do khối lượng tính mà do quét scene trong vòng lặp; thay bằng sổ có sẵn → **~0,36 ms/tick (×14)**, đúng luật B8, không cần Burst. Thành phố 1000 NPC: khung p95 9,4 ms. → **Entities chưa cài.**

## 6. Lên 3D thật (08–13/10)
Vì tọa độ đã là `Vector3Int` và save đã phẳng, bước lên thế giới khối chỉ là *mở rộng tầng thứ ba*, không phải xây lại:
- Hai bộ vẽ 2D (tilemap) và 3D (voxel) là **hai bản vẽ ngang hàng của cùng một dữ liệu**: vị trí là dữ liệu (`CellPos`), sim chỉ phát `Changed`, renderer tự nghe. Có test quét mã nguồn: ngoài renderer cấm gọi vẽ lại; lớp 3D cấm đọc sprite/transform.
- Tách assembly (`Nizhangjie`, `.Editor`, `.VoxelView`) để game không tham chiếu được lớp voxel. Sau đó 2D được "đóng kho" ([case 07](../case-studies/07-the-gioi-khoi-render-anh-sang-nuoc.md)).

## Điều tôi rút ra
- **Chuẩn bị rẻ ở chỗ không sửa muộn được** (định dạng save, API mod) và **hoãn ở chỗ sửa muộn vẫn được** (đổi kiểu toàn engine).
- **Khảo sát bằng số trước khi quyết** (251 file, 267 lỗi) và **dừng khi thấy lớn quá** thay vì đẩy tiếp lúc không ai kiểm.
- **Không theo trào lưu:** chọn DOTS hybrid, đo xong thấy chưa cần mở rộng thì dừng.

## Bằng chứng
[`docs/VECTOR3INT_MIGRATION_NOTES.md`](../docs/VECTOR3INT_MIGRATION_NOTES.md) · [`project_voxel_z_future_proofing.md`](../docs/lessons/project_voxel_z_future_proofing.md) · [`project_dots_hybrid.md`](../docs/lessons/project_dots_hybrid.md) (luật C9 trong [`CODE_SYSTEM_RULE`](../docs/CODE_SYSTEM_RULE.md)) · [`project_render_2d3d_coupling.md`](../docs/lessons/project_render_2d3d_coupling.md)
