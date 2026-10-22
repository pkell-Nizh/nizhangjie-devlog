# Hạ tầng dữ liệu: engine đọc mọi thứ từ JSON, và những bẫy tôi dính

**Thời gian:** nền tảng 22–26/06/2026, mở rộng suốt dự án · **Kỹ năng:** thiết kế kiến trúc dữ liệu, kiểm tra cấu hình, tương thích save.

## Quyết định gốc
Tôi chốt từ tháng 6: *"Nizhangjie là engine, logic tách hoàn toàn khỏi dữ liệu (giống Minecraft/Terraria/RimWorld). Mọi đơn vị, quái, kỹ năng, vật phẩm, vùng đất là nội dung JSON, mod người dùng drive y như game gốc."* Từ đó thành luật A1 "JSON-first" trong [`CODE_SYSTEM_RULE`](../docs/CODE_SYSTEM_RULE.md).

## Những mảnh chính
| Mảnh | Việc nó làm |
|---|---|
| **`GameDatabase`** | điểm gom (composition root): nạp → kiểm tra → khoá (`Freeze`) → tra cứu theo kiểu `Get<T>(id)` |
| **`namespace:id`** | id kiểu Minecraft (`core:beast_wolf`, `pokemon:move_tackle`); namespace tự suy từ thư mục mod. Thêm mới và ghi đè phân biệt tự nhiên: mod tự đặt namespace thì *thêm*, ghi đúng `core:x` thì *sửa* |
| **Newtonsoft.Json (opt-in)** | tôi lật quyết định ban đầu dùng JsonUtility vì cần `Dictionary` và object lồng nhau; kiểu nào gắn `[UseNewtonsoftJson]` mới dùng; lưu ý Newtonsoft không tôn trọng `[NonSerialized]` nên phải `[JsonIgnore]` |
| **`ModLoader`** | sắp thứ tự nạp bằng topo-sort (Kahn), tự loại mod thiếu phụ thuộc hoặc vòng phụ thuộc mà không làm hỏng mod khác |
| **Localization** | text theo mã chuỗi, nạp từ JSON, mod cộng thêm được, có ngôn ngữ dự phòng |
| **Kiểm tra lúc khởi động** | tham chiếu trỏ tới def không tồn tại, giá trị ngoài miền → gom thành **một báo cáo** có chỉ rõ def nào, field nào |
| **`ConfigErrors()`** | luật B7: lưới an toàn nằm ở kiểm tra cấu hình, không ở bước resolve tham chiếu |

**Canary trước khi mở rộng:** chọn một kiểu dữ liệu nhỏ (một loài thú mẫu) làm "chim hoàng yến", **13/13 test xanh trong Unity** ngày 26/06, rồi mới mở `LoadAll` ra 13 kiểu (18/18 test), `ModLoader` 11 ca, Localization có test riêng.

## Các luật và bẫy rút ra
1. **Hai thư mục JSON song song không tự đồng bộ.** Editor đọc `Assets/Data`, bản build đọc `StreamingAssets/Data`. Ngày 29/07 tôi tạo hai file cấu hình chỉ ở một nơi; Editor không đọc được, chỉ log *warning* "File không tồn tại". Vì mỗi cấu hình có giá trị mặc định trùng số với JSON, mọi thứ *trông như chạy đúng* nhưng thật ra chạy bằng giá trị mặc định, JSON chưa từng được đọc. → luật C2: sửa cả hai bản, `diff` để chắc, kiểm console cả `warning` chứ không chỉ `error`.
2. **Enum đã lưu là append-only** (B3), **sentinel phải phân biệt "chưa có" với "bằng 0"** (B4), **vòng đời dữ liệu phải khai tường minh** (B5).
3. **Mỗi bản save phải sống qua nâng cấp.** Mọi snapshot dùng `int x, int y` phẳng, không serialize `Vector2Int` trực tiếp, nên thêm `int z = 0` sau này không làm hỏng save cũ ([bài 03](03-doi-sang-dots-va-3d.md)). Mỗi gói nội dung có **slot save riêng** thay vì thêm field vào `SaveData`; mod bị tắt thì dữ liệu của nó vẫn được giữ, bật lại không mất.
4. **ID số nguyên chỉ tồn tại trong RAM** (dựng sau `Freeze`), còn save luôn lưu `namespace:id`. Nhờ đó thứ tự nạp đổi cũng không vỡ save.
5. **Nội dung số lượng lớn sinh bằng script** (ví dụ 84 vật liệu và 99 trang bị cho kho vật phẩm), đúng định dạng JSON của lõi, rồi nạp và mặc thử thật trong game.
6. **Đo chi phí nạp dữ liệu:** có marker đo `Data.LoadFile/LoadFolder`. Khi người chơi báo "đi bộ vẫn khựng", đo cho thấy **không phải do nạp dữ liệu** (123 lần nạp đều thuộc lúc khởi động, 0 lần khi chơi).

## Một lần tự kiểm tra ranh giới
Một lớp tự sinh asset mỗi lần nạp assembly (vi phạm đúng luật JSON-first) từng gây vòng lặp import vô hạn và làm Unity treo ở 11 GB RAM. Xem [case 13](../case-studies/13-do-tin-cay-luu-tru-va-su-co.md).

## Bằng chứng
[`project_data_registry.md`](../docs/lessons/project_data_registry.md) · [`feedback_data_folder_split.md`](../docs/lessons/feedback_data_folder_split.md) · [`project_cellindices.md`](../docs/lessons/project_cellindices.md) · [`docs/CODE_SYSTEM_RULE.md`](../docs/CODE_SYSTEM_RULE.md) · bài [Modding](04-modding-va-modder.md)
