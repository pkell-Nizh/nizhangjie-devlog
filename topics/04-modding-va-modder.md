# Modding và modder: hai tầng, không Lua, và một bản hướng dẫn cho người lạ

**Thời gian:** quyết định chiến lược 07/09/2026, hướng dẫn mod 30/09/2026 · **Kỹ năng:** thiết kế bề mặt mở rộng, quản lý chi phí bảo trì, tài liệu cho người ngoài.

## Ràng buộc của tôi
Tôi làm một mình và còn một dự án khoa học khác chạy song song, nên **không thể trực ban bảo trì hệ mod hằng ngày**. Mọi quyết định bên dưới đều từ đó.

## Hai tầng, hai mức cam kết
| Tầng | Cách làm | Cam kết |
|---|---|---|
| **Dữ liệu** | thả file JSON vào `Mods/<mod>/Data/` | **Ổn định.** Tên field và tên thư mục là hợp đồng công khai. Không giấu, vì nó là "bề mặt mod miễn phí" gánh bảo trì gần bằng 0. |
| **Móc C#** | đăng ký vào các sổ móc của lõi | **Không hứa gì.** Mod gãy khi tôi refactor là việc của modder. |

**Vì sao không có Lua:** tôi xếp hạng chi phí bảo trì. Lua đứng cuối vì mặt phơi ra (binding) lớn thêm mỗi lần thêm hệ mà engine lớn rất nhanh, vi phạm luật tất định (thứ tự `pairs`, rng, thời gian), và lưu trạng thái Lua khi save là "địa ngục". Việc rẻ nên làm sớm: sổ đăng ký nhất quán, `priority` trong hook, mỗi sổ khai rõ "chọn một" hay "chạy hết", log khi hai mod giành cùng khoá.

## Bản hướng dẫn cho modder
Ngày 30/09 tôi viết [`MODDING_GUIDE.md`](../docs/MODDING_GUIDE.md) (185 dòng) cho người ngoài. Nó cho thấy bề mặt mod thực tế:
- **Cấu trúc thư mục + `mod_info.json`:** `id`, `enabled` (cất mod mà không xoá), `dependencies` (thiếu thì chỉ mod đó bị loại), `loadAfter/loadBefore` (mềm). Mod dính vòng phụ thuộc bị loại cả nhóm với log rõ ràng.
- **Id kiểu Minecraft:** viết id cục bộ, loader tự gắn tiền tố; tham chiếu sang mod khác phải ghi đủ tiền tố và khai trong `dependencies`. Kiểu `Def` thì *thêm mới*, kiểu cũ thì *ghi đè*; hướng dẫn nói cách nhận biết qua log.
- **Kiểm tra lỗi lúc khởi động:** một báo cáo gom mọi lỗi tham chiếu/giá trị ngoài miền. File JSON hỏng chỉ mất đúng file đó, không sập mod.
- **Sổ điều kiện chung:** cùng cú pháp `conditions` cho tiến hoá, trait ẩn, hồi sinh, công cụ thu phục; `fact` + `flag` đủ cho phần lớn mod chỉ viết JSON.
- **Ví dụ thật:** trait ẩn "Sát Tinh" mở khi đạt 100 lần giết; luật hồi sinh *không có trong lõi* (mặc định chết là chết thật), mod bật bằng JSON.
- **Móc C#:** `ConditionRegistry`, hook vòng đời sinh vật, nhà cung cấp dữ kiện, `PackSaveRegistry` (dữ liệu mod có slot save riêng, mod tắt không mất dữ liệu), sự kiện cái chết.
- **Quy ước:** id chữ thường `snake_case`, JSON UTF-8 không BOM, mỗi def một file, **không đặt tên/hình/roster từ thương hiệu có bản quyền vào mod định phát hành**.

## Chứng minh bằng ba gói nội dung thật
Cuối tháng 9, ba gói rất khác nhau cùng sống trên một lõi (xem [case 11](../case-studies/11-mod-va-content-pack.md)):
- **`pokemon`**: sưu tập sinh vật (18 hệ, tính cách, chiêu, đặc tính, huấn luyện viên NPC, Field Codex, shiny), có thể tắt bằng `enabled: false`.
- **`xianxia`**: toàn bộ nội dung tu tiên (thành phố, trận pháp, NPC tu sĩ, Cảnh Giới).
- **`tensura`**: cơ chế Đặt Tên, đã được cất thành mod tắt, kèm ghi chú những việc phải soát khi bật lại.
- **`DevPokemon`**: bộ 149 loài chỉ dùng khi phát triển, **không có trong bản public**.

Tổng cộng khoảng **257 file JSON** nằm trong các mod `pokemon` và `xianxia`, và game lõi vẫn chạy khi gỡ chúng.

## Modder là một phần chiến lược ra mắt
Bề mặt mod không chỉ là kỹ thuật. Tôi coi modder là **lực lượng quảng bá**: mod dựng lại truyện nổi tiếng là nguồn nội dung vô tận, và ý tưởng **mod có thể yêu cầu key bản quyền (tuỳ chọn)** nhằm cho modder quyền kiểm soát, kèm quyền lợi để họ muốn bật. Chi tiết và những phản biện nằm ở [bài Chiến lược ra mắt](01-chien-luoc-ra-mat.md).

## Điều tôi học được
- **Mức cam kết khác nhau cho từng tầng** là cách giữ mod mở mà không phải trực ban.
- **Chứng minh ranh giới bằng cách tách nội dung thật ra khỏi lõi**, không chỉ viết tài liệu.
- **Nói rõ cái gì không hứa** (móc C#) cũng quan trọng như cái gì hứa.

## Bằng chứng
[`docs/MODDING_GUIDE.md`](../docs/MODDING_GUIDE.md) · [`project_modding_strategy.md`](../docs/lessons/project_modding_strategy.md) · [`project_data_registry.md`](../docs/lessons/project_data_registry.md) · [case 11](../case-studies/11-mod-va-content-pack.md) · bài [Hạ tầng dữ liệu](02-ha-tang-du-lieu.md)
