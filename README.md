# Nizhangjie — nhật ký thiết kế & kỹ thuật một game sandbox 3D (Unity / C#)

> **EN summary:** A solo Unity/C# colony-sim + voxel-world game, built with AI pair-programming (Claude Code).
> This repo is *not* the source code. It is the **process**: plans, measurements, mistakes, fixes, decisions and screenshots.

Game kết hợp quản lý khu định cư (kiểu RimWorld), thế giới khối (kiểu Minecraft/Hytale), bắt thú (kiểu Pokémon) và tu tiên.
Mọi nội dung mở rộng được qua **mod dữ liệu JSON + hook C#**.

## Cách mình làm việc (điều mình muốn nhà tuyển dụng thấy)
1. **Lập kế hoạch trước, bằng văn bản** → [`docs/`](docs) (mỗi hệ lớn có một file thiết kế: trồng trọt, thời tiết, combat, shop, sự kiện thế giới...).
2. **Đo trước khi tối ưu** → số liệu Profiler thật (ms/khung, KB GC), không đoán.
3. **Ghi lại lỗi và cách sửa** → [`docs/lessons/`](docs/lessons): nguyên nhân gốc, cái đã thử mà tệ hơn, bẫy khi đo.
4. **Chứng minh bằng ảnh chụp trong game** → thư mục [`images/`](images).
5. **Kiến trúc có luật**: dữ liệu (JSON) tách khỏi hành vi (C#), hệ lõi tách khỏi nội dung tu tiên, mỗi hệ có test.

## Ba câu chuyện tiêu biểu

### 1. Địa hình 3D bị "cột đá vô tri" — đo, tìm nguyên nhân, sửa
- **Triệu chứng:** người chơi thấy nhiều cột đá/cỏ vô nghĩa trên thế giới.
- **Giả thuyết ban đầu sai:** tưởng hàm độ cao lỗi. Đo thật trong Play Mode: *không* phải.
- **Nguyên nhân gốc:** bước điêu khắc 3D khoét 5–7 khối quanh một điểm, để lại mặt đất gốc thành cột (34/37 trường hợp).
- **Bẫy khi đo:** lần đầu đếm ra 525 "cột"/ô, thật ra là *cây*. Phải loại khối cây khỏi phép đo.
- **Sửa:** giới hạn điêu khắc theo độ cao đất lân cận → còn 0 cột nhô.
- Chi tiết: [`docs/lessons/project_sculpt_pillars_fix.md`](docs/lessons/project_sculpt_pillars_fix.md) · Ảnh: [`images/worldgen/`](images/worldgen) (có các cặp **cũ vs mới**).

### 2. NPC "tới được" nhưng không tìm được đường (CanReach = true, FindPath = null)
- 87/252 cặp điểm bị lệch. Kiểm tra giả thuyết "hệ region nói dối" → **sai**, region đúng.
- Nguyên nhân thật: A* hành lang 3D chạm trần số nút trên đường rất vòng; một phần nhỏ do ô công trình bị coi là vật cản.
- Sửa heuristic A* (kết hợp khoảng cách đồ thị region), thêm test hồi quy (đỏ trên code cũ) → lệch **87 → 11**.
- Đã thử một heuristic khác, **tệ hơn** trên núi → bỏ, ghi lại lý do.
- **Bẫy đo:** đo khi map chưa nạp xong cho số đẹp giả. Luôn so cũ/mới trong cùng một phiên.
- Chi tiết: [`docs/lessons/project_canreach_findpath_gap.md`](docs/lessons/project_canreach_findpath_gap.md)

### 3. Hiệu năng: từ "FPS trung bình ổn" đến "hết giật cục"
- Profiler cho thấy một hàm quét mỗi khung ăn 4,1 ms → chuyển sang quét theo sự kiện/0,15 s ⇒ **0,5 ms**.
- Nhận ra FPS trung bình đánh lừa: đo tiếp **giật cục** (p95, đập khối, qua ranh ô). Tìm ra dựng lại lưới đồng bộ, A* NPC trên luồng chính...
- Trung thực với kết quả: p95 *xấu hơn* sau lần sửa đầu và mình ghi rõ "chưa rõ nguyên nhân" thay vì tuyên bố thành công.
- Chi tiết: [`docs/lessons/project_fps_profile_2026-10-14.md`](docs/lessons/project_fps_profile_2026-10-14.md), [`project_hitch_profile_2026-10-15.md`](docs/lessons/project_hitch_profile_2026-10-15.md), [`project_walk_hitch_2026-10-17.md`](docs/lessons/project_walk_hitch_2026-10-17.md)

## Các quyết định thiết kế đáng kể
- **Viết lại bộ sinh thế giới cho 3D liền** (trục đứng không giới hạn, sông chảy thật, hồ sơ biome bằng JSON): [`project_worldgen_3d_redesign.md`](docs/lessons/project_worldgen_3d_redesign.md)
- **Modding: dữ liệu JSON ⟂ hook C#, không dùng Lua**: [`project_modding_strategy.md`](docs/lessons/project_modding_strategy.md)
- **Kiểm toán code "spaghetti"** và dọn kiến trúc: [`docs/AUDIT_SPAGHETTI.md`](docs/AUDIT_SPAGHETTI.md)
- **Thiết kế trước khi code** cho trồng trọt, thời tiết, nhiệt độ, combat, shop, sự kiện thế giới: xem [`docs/`](docs)

## Ảnh chụp
| Chủ đề | Thư mục |
|---|---|
| Tiến hoá từ prototype 2D → thế giới khối liền (18 ảnh có đánh số theo thứ tự) | [`images/prototype`](images/prototype) |
| Sinh địa hình 3D, so sánh cũ/mới, sông, hồ, núi, rừng | [`images/worldgen`](images/worldgen) |
| Hệ ánh sáng khối (đuốc, hang, phòng tối) | [`images/lighting`](images/lighting) |
| Chất lỏng (nước chảy, hồ, dung nham) | [`images/fluids`](images/fluids) |
| Thành phố, công trình, nông trại | [`images/city`](images/city) |
| UI/HUD, màn hình chính, quản lý thú | [`images/ui`](images/ui) |

![Thế giới khối liền](images/prototype/12_the_gioi_lien_toan_canh.png)
![Ánh sáng khối](images/lighting/light_overview.png)

## Công cụ
Unity · C# · Burst/Jobs (DOTS hybrid) · A* · JSON data-driven mods · Unity Profiler · Claude Code (AI pair-programming, kiểm chứng bằng test và số đo).

*Ghi chú: các file trong `docs/lessons/` là ghi chú làm việc gốc (tiếng Việt), giữ nguyên để thấy quá trình thật.*
