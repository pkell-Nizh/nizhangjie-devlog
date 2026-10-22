# Case study 11 — Kiến trúc mod: "dữ liệu JSON ⟂ hook C#", loại bỏ Lua, và chứng minh bằng cách tách 3 gói nội dung ra khỏi lõi

**Thời gian:** quyết định 07/09/2026; hiện thực 26/09 → 03/10 · **Kỹ năng:** thiết kế kiến trúc mở rộng, phân tích chi phí bảo trì, ranh giới module có test.

## Bối cảnh ràng buộc
Người làm là **solo dev**, còn một dự án khoa học (robotics) song song → *không muốn trực ban bảo trì hệ mod hằng ngày*. Nền kỹ thuật đã có từ 26/06: `GameDatabase`, định danh `namespace:id`, nạp bằng Newtonsoft, một "canary" (một loài thú mẫu) với **13/13 test xanh trong Unity**.

## Quyết định: tách 2 tầng, mỗi tầng một mức cam kết
| Tầng | Cam kết | Lý do |
|---|---|---|
| **Dữ liệu** (JSON def, thư mục, `namespace:id`, schema) | ship kèm game, đọc trần, **không giấu** — modder học bằng cách đọc file | gánh bảo trì ~0: thêm data không đụng code |
| **Hook C#** (`IXContributor`, registry, interface) | *at-your-own-risk*: không công bố API, không doc, không hứa ổn định | nếu hứa thì phải on-call |

### Xếp hạng chi phí bảo trì → loại Lua
JSON def + `namespace:id` + validate + topo-sort **~0** → DSL điều kiện nhỏ đóng (~10 toán tử) rất thấp → hook C# không hứa gì thấp → Harmony + hợp đồng ổn định cao → **Lua/VM nhúng cao nhất — đã loại**, vì (1) binding surface lớn thêm *mỗi lần* thêm hệ mà engine lớn rất nhanh (66+ mốc), (2) đụng luật tất định (`pairs` order, rng, `os.time`), (3) serialize trạng thái Lua khi save là "địa ngục".

### Việc rẻ nên làm sớm
- Registry nhất quán, `namespace:id` kỷ luật (add vs override tường minh), validate fail-fast có *provenance*, `ModLoader` topo-sort, cô lập try/catch theo gói.
- Thêm `priority` vào interface hook base; mỗi registry khai rõ **pick-one vs run-all** + log khi mod thứ hai giành cùng key.
- (Tuỳ chọn) một cây predicate + chuỗi effect bằng JSON để sửa logic quest/event/evolution không cần recompile Unity (một lần reimport từng mất 10+ phút).

## Hiện thực — lịch sử commit 26/09 → 03/10
Đây là bằng chứng ranh giới *thật*:
- 26/09 hệ sưu tập sinh vật đầu tiên "như một mod": từ điển thú, **điều kiện tiến hoá dữ liệu hoá** (item, shiny); bắt thú trong trận bằng túi.
- 28/09 **cất Tensura thành mod tắt**; sổ điều kiện chung + một quy ước giới tính cho mọi pack; **đưa tiến hoá vào lõi** để mọi pack dùng chung; **đóng gói nội dung Pokémon thành mod `pokemon`**; pack có *slot save riêng* thay vì field trong `SaveData`; vòng đời sinh vật/sự kiện/diện mạo đi qua **hook registry của lõi**; tách driver theo lượt thành *khung lõi + bộ luật Pokémon*; sinh vật mang *danh sách id diện mạo* thay vì cờ shiny.
- 29/09 "**kéo nốt code pack ra khỏi lõi và canh biên giới**" (test).
- 30/09 viết **hướng dẫn mod cho guest mod**; cất nội dung chỉ-tu-tiên thành mod `xianxia` (thành phố, trận pháp, kịch bản tông phái, NPC tu sĩ, dòng Cảnh Giới trong UI qua hook lõi).
- 03/10 chuyển nội dung thành phố cổ điển sang mod xianxia và thêm bộ công trình Pokémon.

**Điều này chứng minh gì:** khi gỡ nội dung cụ thể (xianxia/Tensura/Pokémon), game lõi vẫn chạy; ba gói nội dung rất khác nhau (tu tiên / slime-naming / sưu tập sinh vật) cùng sống trên một lõi.

## Một chi tiết trung thực
Trong ghi chú Tensura có dòng "khi bật lại cần soát: race id so chuỗi thô, `RaceOps` fallback `core:`, `ThoughtRegistry` khoá theo `d.id` thô" — tức mình ghi rõ *nợ còn lại khi bật lại*, thay vì giả vờ cất gọn.

## Bằng chứng
[`project_modding_strategy.md`](../docs/lessons/project_modding_strategy.md) · [`project_data_registry.md`](../docs/lessons/project_data_registry.md) · [case study 05](05-tach-loi-khoi-tu-tien.md)
