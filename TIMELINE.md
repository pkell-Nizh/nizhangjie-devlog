# Dòng thời gian dự án (26/05/2026 → 23/10/2026)

> **Nguồn dữ liệu, nói rõ để trung thực:** lịch sử git chỉ bắt đầu từ **12/09/2026** (trước đó dự án chưa dùng git).
> Các mốc tháng 5–8 lấy từ **ngày sửa file thiết kế/README** và **ghi chú làm việc** mình viết lúc đó (`docs/early/`, `docs/lessons/`).
> Từ 12/09 trở đi có thêm commit thật. Ảnh chụp có sẵn từ **03/08**, tên file chứa ngày giờ (`images/timeline/`).

## Giai đoạn 1 — Nền móng & thiết kế (26/05 – 06/06)
**Mục tiêu:** viết "Project Bible" trước khi code: game quản lý môn phái tu tiên, UI kiểu RimWorld, kiến trúc dữ liệu JSON.
- 26/05: [Project Bible + Gameplay Design](docs/early/XianXian_Gameplay_Design.md); M1 EventBus + ServiceLocator ([M1](docs/early/M1_README.md)).
- 26–28/05: M3 Combat, M4 mô phỏng, M5 hệ thống sim, M6 chiều sâu nhân vật, M7–M9 chế tác/trang bị. Mỗi mốc có một README ghi cấu trúc + cách dùng. Tổng quan: [PROJECT_SUMMARY](docs/early/PROJECT_SUMMARY.md).
- **Quyết định kiến trúc ngay từ đầu:** JSON là nguồn sự thật, không singleton (dùng ServiceLocator), mọi hệ giao tiếp qua EventBus, code là class thuần để test được.
- 01/06: [Phân tích UI/UX toàn diện](docs/early/UIUX_Analysis.md) trước khi làm giao diện.
- 07/06: **M10 làm lại sinh bản đồ** theo prototype FastNoiseLite, pipeline 9 bước ([M10 redesign](docs/early/M10_REDESIGN_FASTNOISE.md), [pipeline](docs/early/M10_WORLDGEN_PIPELINE.md)).

## Giai đoạn 2 — Dữ liệu hoá & mod (13/06 – 04/07)
- 13/06: hệ kiếm tiền/nghề; thiết kế NPC.
- 23/06: thiết kế hệ Yêu Thú kiểu Pokémon.
- **26/06: nền tảng data-driven/modding** (GameDatabase, `namespace:id`, nạp bằng Newtonsoft). Thử trên 1 loài thú mẫu, **13/13 test xanh trong Unity**. → [bài học](docs/lessons/project_data_registry.md). Cùng ngày viết [tài liệu bàn giao](docs/early/HANDOVER_ANTIGRAVITY.md) cho công cụ AI khác (dự án đã làm với nhiều công cụ AI).
- 29/06: chốt thiết kế M13 (colony-sim thật): [quyết định + 4 phase](docs/lessons/project_m13_gameplay_design.md).
- 04/07: **viết lại sinh thành phố** kiểu "thành quách" (trục chính, chia khối BSP, hẻm, nhà dãy) → [ghi chú](docs/lessons/project_citygen_wallcity.md).

## Giai đoạn 3 — Hệ Pawn/Job kiểu RimWorld (07/07 – 31/07)
- 07/07: [thiết kế lại combat](docs/lessons/project_combat_redesign.md).
- 11–16/07: [masterplan hệ Pawn](docs/lessons/project_pawn_system_masterplan.md): lệnh chỉ định, WorkGiver, lịch, thu hoạch, xây, khuân vác; AI "não" ThinkTree/JobGiver; [Job Pipeline](docs/lessons/project_job_pipeline.md) (JobDriver/Toil).
- 24–25/07: M18–M21 (kéo thả, trạng thái, nhàn rỗi, phản ứng đe doạ, trồng cây). Quy trình: **code → kiểm trong Play Mode → user xác nhận** (ghi rõ ngày verify 14/08).
- 28/07: ngày/đêm; đồng hồ toàn cục (test xanh); vòng đời tài nguyên.
- 30/07: M26 sản xuất, M27 thời khoá biểu, bảng token UI.

## Giai đoạn 4 — Thành phố sống & hiệu năng 2D (01/08 – 12/08)
- 01/08: **lag city 2,8 → 92,6 FPS** nhờ gộp tilemap theo chunk 16×16 (mỗi ô một draw call là thủ phạm) → [ghi chú](docs/lessons/project_tilemap_chunk_mode.md). Công trình thành "nhà thật" (tường/phòng/cửa).
- 02/08: 1000 NPC thành phố đi làm/về nhà.
- 03/08: [chốt nhịp thời gian](docs/lessons/project_time_pacing.md) (ngày 18 phút thật, 30 TPS). **Ảnh chụp đầu tiên:** `images/timeline/screenshot-20260803-*`.
- 08/08: [sửa lag giờ đi làm](docs/lessons/project_city_commute_perf.md) (A* quét nhà dân). Ghi thành luật: *đo trước khi tối ưu*.

## Giai đoạn 5 — Chiều sâu mô phỏng (13/08 – 21/08)
- 13–15/08: sức khoẻ/chấn thương (M30), suy sụp tinh thần (M32), đau buồn (M35), xác chết (M36), ô trang bị theo dữ liệu (M34).
- **15/08: quyết định kiến trúc lớn M37** — tách *Core life-sim* khỏi *Xianxia* để tu tiên thành mô-đun gỡ được → [M37](docs/lessons/project_m37_core_xianxia_split.md).
- 17–20/08: tương tác pawn theo **quan hệ**, màn hội thoại kiểu visual novel, đổi chác vs mua bán.
- **21/08: hệ trí nhớ & lời thoại (M48–M54)** — sử ký → ký ức chủ quan → ghép mảnh câu. Có số đo: **lặp lời 33,1% → 0,0%**; câu mặt 70 → 564. Test bắt được bug thiết kế bị che. → [M50](docs/lessons/project_m50_speech_quality.md), [M54](docs/lessons/project_m54_speech_audit.md).

## Giai đoạn 6 — Gộp hệ thống & sự cố (26/08 – 29/08)
- 26/08: viết [CODE_SYSTEM_RULE](docs/CODE_SYSTEM_RULE.md): bộ luật mà **mỗi luật gắn với một bug thật**; lập catalogue tính năng.
- 27/08: [sự cố HUD.prefab](docs/lessons/project_hud_prefab_unify.md) — hai scene dùng chung prefab đè lẫn nhau; khôi phục thành thiết kế hợp nhất có chủ đích.
- 28/08: pawn di chuyển giữa nhiều ô thế giới (M58–M61), lưu/khôi phục công trình.
- 29/08: [Unity treo + RAM 11 GB](docs/lessons/project_editor_open_hang_2026-08-29.md) — điều tra và sửa nguyên nhân.

## Giai đoạn 7 — Chuyển sang thế giới khối 3D (24/08 – 03/10)
- Ảnh theo ngày: `images/timeline/` (24/08 → 17/09) cho thấy từ 2D sang khối.
- **12–17/09 (có commit git):** migrate toàn engine `Vector2Int → Vector3Int`; prototype xây nhà nhiều tầng. Chuỗi commit cho thấy **quá trình thử–sai thật**: nhiều lần *Revert → Reapply → Revert* ("giới hạn cỡ phòng phụ", "gộp ô tường chết", "bếp tách rời") và cuối cùng **viết lại thuật toán** — từ tối ưu Metropolis/simulated annealing sang **chia BSP/guillotine (0% ô thừa theo cấu trúc)**, rồi "partition cho hình học + spanning tree cho lối đi".
- 18/09: bắt đầu hệ biome + địa hình theo chunk.
- Prototype voxel 18 ảnh có đánh số: [`images/prototype`](images/prototype).

## Giai đoạn 8 — Thế giới như Minecraft (04/10 – 23/10)
Chi tiết ở [README](README.md): viết lại sinh địa hình 3D, hang/nước/dung nham, ánh sáng khối, đường đi nhiều tầng, màn hình chính, đo & sửa giật cục, túi đồ 2D kiểu Minecraft, và hệ **trồng trọt** (6 bước, mỗi bước một commit rõ ràng; bước 7 hoãn có chủ đích).

## Mẫu hình lặp lại (điều mình rút ra)
1. **Thiết kế bằng văn bản trước, code sau** — mỗi hệ có file thiết kế.
2. **Đo rồi mới sửa** — luôn kèm con số trước/sau.
3. **Báo cáo trung thực** — ghi "chưa verify", "chưa rõ nguyên nhân", thử cái tệ hơn thì bỏ và ghi lại.
4. **Mỗi bug thành một luật** trong CODE_SYSTEM_RULE để không lặp lại.
5. **Tách lõi ⟂ nội dung** (M37) và **dữ liệu ⟂ hành vi** để mở rộng/mod được.
