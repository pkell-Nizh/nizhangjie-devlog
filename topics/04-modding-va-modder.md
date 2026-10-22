# Modding và modder: hai tầng, không Lua, và một bản hướng dẫn cho người lạ

**Thời gian:** quyết định chiến lược 07/09/2026, hướng dẫn mod 30/09/2026 · **Kỹ năng:** thiết kế bề mặt mở rộng, quản lý chi phí bảo trì, tài liệu cho người ngoài.

## Ràng buộc của tôi
Tôi làm một mình và còn một dự án khoa học khác chạy song song, nên **không muốn trực ban bảo trì hệ mod hằng ngày**. Mọi quyết định dưới đây xuất phát từ đó.

## Nguyên tắc gốc: tách hai tầng, mỗi tầng một mức cam kết
Câu "không doc, không hứa, không trực ban" chỉ áp cho **tầng móc C#**, *không áp* cho **tầng dữ liệu**.

| Tầng | Cách làm | Cam kết |
|---|---|---|
| **Dữ liệu** | thả JSON vào `Mods/<mod>/Data/` | **Ổn định.** Tên field và thư mục là hợp đồng công khai. Không giấu: nó tự lộ cấu trúc (có chủ ý), gánh bảo trì ~0, modder học bằng cách đọc file ship sẵn. |
| **Móc C#** | đăng ký vào các sổ móc của lõi | **Không hứa.** Mod gãy khi tôi refactor là việc của modder. |

### Xếp hạng chi phí bảo trì (vì sao loại Lua)
JSON + `namespace:id` + kiểm tra + topo-sort **~0** → DSL điều kiện nhỏ đóng (~10 toán tử) *rất thấp* → hook C# không hứa gì *thấp* → Harmony + hợp đồng ổn định *cao* → **Lua/VM nhúng cao nhất, đã loại.** Lý do: mặt phơi ra lớn thêm *mỗi lần* thêm hệ mà engine lớn rất nhanh (66+ mốc); đụng luật tất định (thứ tự `pairs`, rng, thời gian); lưu trạng thái Lua khi save là "địa ngục".

### Danh sách "không làm" tôi ghi lại
Không thêm Lua; không chính thức hỗ trợ Harmony và không ký hợp đồng hook ổn định; không công bố API của tầng hook. **Harmony chỉ là cửa thoát hiểm**: mỗi lần có người phải dùng nó là tín hiệu thiếu hook ở đó, nhưng gom theo nhịp phát hành chứ không vá ngay.

### Mô hình tôi học từ trò khác
- **Đường đăng ký vào điểm mở rộng (`IXContributor`), không monkey-patch.** So với Factorio: nuốt hàng trăm mod nhờ cấm monkey-patch và thứ tự nạp tường minh, không nhờ Lua.
- **Mô hình Minecraft làm đường thoát cuối:** "ship gì đâu, hứa gì đâu" vẫn ra hệ sinh thái khổng lồ. Thứ tôi "nợ" để đường đó mở: **đừng chủ động chống lại** (không làm rối mã nặng, không chống can thiệp, giữ assembly nạp được).
- **Khi cộng đồng mạnh:** họ tự đủ sức dựng API; lúc đó **không tự dựng từ đầu mà "ban phước" cho loader họ đã dựng**, thêm vài hook chính thức. Chất lượng điểm nối bây giờ quyết định cộng đồng hội tụ về *một* loader hay đẻ ba loader xung khắc (kiểu Forge/Fabric/Rift).

### Việc rẻ nên làm sớm
Sổ đăng ký nhất quán; `namespace:id` kỷ luật; `priority` trong interface hook; mỗi sổ khai rõ "chọn một" hay "chạy hết" và log khi hai mod giành cùng khoá; mỗi hệ có một `…HandlerRegistry`. Tuỳ chọn: một cây điều kiện bằng JSON để sửa logic không cần biên dịch lại (một lần reimport từng mất 10+ phút) — **cuối tháng 9 tôi làm một phần việc này thành "sổ điều kiện chung" (28/09).**

## Bản hướng dẫn cho modder (30/09)
[`MODDING_GUIDE.md`](../docs/MODDING_GUIDE.md) (185 dòng) viết cho người ngoài, *không mâu thuẫn* với nguyên tắc trên: phần dữ liệu cam kết ổn định, phần móc C# ghi rõ "không hứa". Nội dung chính:
- **Cấu trúc + `mod_info.json`:** `id`, `enabled` (cất mod mà không xoá), `dependencies` (thiếu thì chỉ mod đó bị loại), `loadAfter/loadBefore` (mềm). Mod dính vòng phụ thuộc bị loại cả nhóm với log rõ.
- **Id kiểu Minecraft:** viết id cục bộ, loader tự gắn tiền tố; tham chiếu sang mod khác phải ghi đủ tiền tố và khai `dependencies`. Kiểu `Def` thì *thêm*, kiểu cũ thì *ghi đè*; hướng dẫn nói cách nhận biết qua log.
- **Kiểm tra lúc khởi động:** một báo cáo gom mọi lỗi. File JSON hỏng cú pháp chỉ mất đúng file đó.
- **Sổ điều kiện chung:** cùng cú pháp `conditions` cho tiến hoá, trait ẩn, hồi sinh, công cụ thu phục; `fact` + `flag` đủ cho phần lớn mod chỉ viết JSON.
- **Ví dụ thật:** trait ẩn "Sát Tinh" mở khi đạt 100 lần giết; luật hồi sinh *không có trong lõi* (mặc định chết là chết thật), mod bật bằng JSON.
- **Móc C#:** `ConditionRegistry`, hook vòng đời sinh vật, nhà cung cấp dữ kiện, `PackSaveRegistry` (dữ liệu mod có slot save riêng, mod tắt không mất dữ liệu), sự kiện cái chết.
- **Quy ước:** id chữ thường `snake_case`, JSON UTF-8 không BOM, mỗi def một file, **không đặt tên/hình/roster từ thương hiệu có bản quyền vào mod định phát hành.**

## Chứng minh bằng gói nội dung thật
Cuối tháng 9 (xem [case 11](../case-studies/11-mod-va-content-pack.md)) nội dung cụ thể được tách khỏi lõi:
- **`pokemon`**: sưu tập sinh vật (18 hệ, tính cách, chiêu, đặc tính, huấn luyện viên NPC, Field Codex, shiny); tắt bằng `enabled: false`.
- **`xianxia`**: nội dung tu tiên (thành phố, trận pháp, NPC tu sĩ, Cảnh Giới).
- **`tensura`**: cơ chế Đặt Tên, đã cất thành mod tắt kèm ghi chú việc phải soát khi bật lại.
- **`DevPokemon`**: bộ 149 loài chỉ dùng khi phát triển, **không có trong bản public**.

Tổng cộng khoảng **257 file JSON** trong `pokemon` và `xianxia`; game lõi vẫn chạy khi gỡ chúng.

## Modder là một phần chiến lược ra mắt
Tôi coi modder là **lực lượng quảng bá**: mod dựng lại truyện nổi tiếng là nội dung vô tận, và ý tưởng **mod có thể đòi key bản quyền (tuỳ chọn)** cho modder quyền kiểm soát, kèm quyền lợi để họ muốn bật. Chi tiết và phản biện: [Chiến lược ra mắt](01-chien-luoc-ra-mat.md). Claude khi đó cũng nhắc rằng chiến lược này **hứa một bề mặt mod cỡ Cobblemon ngay lúc ra mắt**, tức tầng mod phải có thật chứ không phải lời hứa.

## Điều tôi học được
- **Mức cam kết khác nhau cho từng tầng** là cách giữ mod mở mà không phải trực ban.
- **Chứng minh ranh giới bằng cách tách nội dung thật ra khỏi lõi**, không chỉ viết tài liệu.
- **Nói rõ cái gì không hứa** cũng quan trọng như cái gì hứa.

## Bằng chứng
[`docs/MODDING_GUIDE.md`](../docs/MODDING_GUIDE.md) · [`project_modding_strategy.md`](../docs/lessons/project_modding_strategy.md) · [`project_data_registry.md`](../docs/lessons/project_data_registry.md) · [case 11](../case-studies/11-mod-va-content-pack.md) · bài [Hạ tầng dữ liệu](02-ha-tang-du-lieu.md)
