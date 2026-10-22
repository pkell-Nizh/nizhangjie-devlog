# Hạ tầng dữ liệu: engine đọc mọi thứ từ JSON, và những bẫy tôi dính

**Thời gian:** thiết kế 22/06, canary 26/06/2026, mở rộng suốt dự án · **Kỹ năng:** thiết kế kiến trúc dữ liệu, kiểm tra cấu hình, tương thích save.

## Quyết định gốc
Tôi chốt từ tháng 6: *Nizhangjie là engine; logic tách hoàn toàn khỏi dữ liệu (giống Minecraft/Terraria/RimWorld). Mọi đơn vị, quái, kỹ năng, vật phẩm, vùng đất là nội dung JSON, mod người dùng drive y như game gốc.* Từ đó thành luật A1 "JSON-first" với một **phép thử**: *modder thêm một biome/trang bị/công thức mới mà không mở Visual Studio — nếu không được, chỗ đó chưa data-driven.*

## Sáu quyết định chốt (22/06)
1. `GameDatabase` làm điểm gom lúc khởi động (composition root).
2. Tra cứu theo kiểu `Get<T>(id)` + **`namespace:id`** (`core:beast_wolf`); namespace tự suy từ tên thư mục mod.
3. Namespace phân biệt **thêm mới** và **ghi đè** một cách tự nhiên (mod tự đặt namespace thì thêm, ghi đúng `core:x` thì sửa). Cơ chế "patch" kiểu xpath **hoãn lại**.
4. Lớp `Def` trừu tượng thay giao diện `IDefinition`: có id/tên/mô tả, được loader đóng dấu nguồn, và có hai hook kiểm tra (`ResolveReferences`, `ConfigErrors`).
5. **Bước kiểm tra lúc khởi động thay cho "compile check" của dữ liệu:** tham chiếu thiếu/sai kiểu/sai enum thì chặn khởi động và báo rõ mod nào.
6. Sinh hằng số id bằng codegen: tuỳ chọn, để sau.

> **Điều tôi hiểu sai và được giải thích lại:** tôi tưởng `namespace:id` giúp "tìm lỗi lúc biên dịch". Không: chuỗi không đi qua trình biên dịch. Lợi ích thật là *chống va chạm giữa mod*, *thất bại sớm lúc nạp*, và *biết lỗi do mod nào*.

## Các mảnh khác
- **Newtonsoft.Json (opt-in):** tôi lật quyết định ban đầu dùng JsonUtility vì cần `Dictionary` và object lồng nhau; kiểu nào gắn `[UseNewtonsoftJson]` mới dùng. Bẫy: Newtonsoft không tôn trọng `[NonSerialized]`, phải `[JsonIgnore]`.
- **`ModLoader`:** sắp thứ tự nạp bằng topo-sort, tự loại mod thiếu phụ thuộc hoặc vòng phụ thuộc mà không làm hỏng mod khác.
- **Localization:** text theo mã chuỗi nạp từ JSON, mod cộng thêm được, có ngôn ngữ dự phòng.
- **Canary trước khi mở rộng:** một kiểu dữ liệu nhỏ (loài thú mẫu) **13/13 test xanh** trong Unity ngày 26/06, rồi mở `LoadAll` ra 13 kiểu (18/18 test), `ModLoader` 11 ca.
- **Dọn logic hardcode:** các chỗ so chuỗi như `entityKind=="beast"` hay `case "bleed"` phải chuyển sang đọc từ dữ liệu; 9 hiệu ứng trạng thái được khai thành JSON để thay dần cho `switch` cứng.

## Bẫy thật đã dính (mỗi cái thành luật)
1. **Chỉ số mặc định im lặng sai (22/06).** Rà `CharacterStats` thì công thức `(base+Σflat)×(1+Σ%cộng)×Π(1+%nhân)` đúng, và nghi ngờ "kháng tính sai" của tôi **không phải bug** (đã kiểm chứng, bỏ nghi ngờ). Chỗ nguy hiểm thật là *không có giá trị mặc định*: nếu nhà máy tạo đơn vị quên đặt `CritDMG`, hàm lấy trả 0 chứ không phải null, nên đòn chí mạng nhân 0 → sát thương 0, mà guard `?? 1.5` không bắt được. → ghi nhận cần bảng mặc định từ dữ liệu.
2. **Thiếu tiền tố namespace im lặng (B7).** JSON khai `"small_talk"` thiếu `core:` → tra cứu trả null *không lỗi, không cảnh báo* → pawn vẫn đi, vẫn đứng, nhưng **không bao giờ** áp độ thiện cảm. `ResolveReferences` chỉ bắt id gõ sai hẳn vì nó tự rơi về `core:`; nên lưới an toàn phải là `ConfigErrors()`. Dữ liệu hỏng thì `Build()` trả null, **không dựng một nửa**. Một test đỏ vì chính lỗi này tồn tại nhiều ngày mà không ai chạy lại (xem [case 05](../case-studies/05-tach-loi-khoi-tu-tien.md)).
3. **Hai thư mục JSON song song không tự đồng bộ (C2, 29/07).** Editor đọc `Assets/Data`, bản build đọc `StreamingAssets/Data`. Tôi tạo hai file cấu hình chỉ ở một nơi; Editor chỉ log *warning*, và vì cấu hình có giá trị mặc định trùng số với JSON, mọi thứ *trông như chạy đúng* nhưng thật ra JSON chưa từng được đọc. Luật: file mới tạo ở `Assets/Data`, sửa file cũ thì sửa cả hai và `diff`, kiểm console cả `warning`. Ngoại lệ đúng chỗ là `StreamingAssets/Mods/`.
4. **Save phải sống qua nâng cấp.** Snapshot dùng `int x, y` phẳng; mỗi gói nội dung có **slot save riêng** thay vì thêm field vào `SaveData`; mod bị tắt thì dữ liệu của nó vẫn được giữ.
5. **ID số nguyên chỉ ở RAM** (dựng sau `Freeze`), save luôn lưu `namespace:id`.
6. **Nội dung số lượng lớn sinh bằng script** (84 vật liệu, 99 trang bị), đúng định dạng JSON của lõi, rồi nạp và mặc thử thật trong game.
7. **Đo trước khi nghi ngờ nạp dữ liệu:** khi báo "đi bộ vẫn khựng", marker `Data.LoadFile/LoadFolder` cho thấy **không phải do nạp dữ liệu** (123 lần nạp đều lúc khởi động, 0 lần khi chơi).

## Một lần dữ liệu tự làm hỏng Editor
Một lớp tự sinh asset mỗi lần nạp assembly (vi phạm đúng luật JSON-first) gây vòng lặp import vô hạn, Unity treo ở 11 GB RAM. Xem [case 13](../case-studies/13-do-tin-cay-luu-tru-va-su-co.md).

## Bằng chứng
[`project_data_registry.md`](../docs/lessons/project_data_registry.md) · [`feedback_data_folder_split.md`](../docs/lessons/feedback_data_folder_split.md) · [`project_cellindices.md`](../docs/lessons/project_cellindices.md) · [`docs/CODE_SYSTEM_RULE.md`](../docs/CODE_SYSTEM_RULE.md) (A1, B7, C2) · bài [Modding](04-modding-va-modder.md)
