# Dòng thời gian dự án (26/05/2026 → 23/10/2026)

> **Về nguồn dữ liệu (nói rõ để trung thực):**
> - Lịch sử **git chỉ bắt đầu từ 12/09/2026** (trước đó dự án chưa dùng git). Từ 12/09 có **248 commit** thật; các mốc sau đó dẫn tới commit cụ thể.
> - Các mốc **tháng 5–8** dựng từ **ngày sửa file thiết kế/README** và **ghi chú làm việc** viết lúc đó (`docs/early/`, `docs/lessons/`).
> - **Ảnh chụp** có sẵn từ 03/08/2026 (tên file chứa ngày giờ, trong `images/timeline/`). Không có ảnh chụp từ tháng 5–7.

**Quy ước độ chắc chắn:** 🟢 có commit/ảnh/số đo · 🟡 chỉ có ghi chú/ngày sửa file.

---

## Giai đoạn 1 — Thiết kế trước khi code (26/05 → 06/06) 🟡
**Mục tiêu:** một "Project Bible" cho game quản lý môn phái tu tiên kiểu RimWorld, kiến trúc dữ liệu bằng JSON.
- **26/05** — [Project Bible + Gameplay Design](docs/early/XianXian_Gameplay_Design.md) (Phần I gameplay, Phần II kiến trúc kỹ thuật); **M1** EventBus + ServiceLocator ([README](docs/early/M1_README.md)).
- **26–28/05** — M3 Combat, M4 mô phỏng, M5 hệ thống sim, M6 chiều sâu đệ tử, M7–M9 chế tác & trang bị ([M3](docs/early/M3_README.md) … [M9](docs/early/M9_README.md)). Tổng quan: [PROJECT_SUMMARY](docs/early/PROJECT_SUMMARY.md).
- **Quyết định kiến trúc ngày đầu:** *JSON là nguồn sự thật* (không ScriptableObject), *không singleton* (ServiceLocator), *mọi hệ giao tiếp qua EventBus*, *class thuần để test được*, **string key thay enum** cho loại tài nguyên/quest → mod được.
- **01/06** — [Phân tích UI/UX toàn diện](docs/early/UIUX_Analysis.md) trước khi làm giao diện.
- **07/06** — **M10 làm lại sinh bản đồ** theo prototype FastNoiseLite, pipeline 9 pass ([redesign](docs/early/M10_REDESIGN_FASTNOISE.md), [pipeline](docs/early/M10_WORLDGEN_PIPELINE.md)).

## Giai đoạn 2 — Dữ liệu hoá & nền tảng mod (13/06 → 04/07) 🟡
- **13/06** hệ kiếm tiền/nghề; **16/06** [thiết kế lại combat](docs/lessons/project_combat_redesign.md) (giữ RTwP có sẵn, bỏ hệ phụ, GridManager singleton bị loại vì vi phạm luật; công thức AOE tuyến tính giảm dần; rng tất định).
- **23/06** thiết kế hệ Yêu Thú kiểu Pokémon.
- **26/06** **nền tảng data-driven/mod**: `GameDatabase`, `namespace:id`, Newtonsoft; canary 1 loài thú **13/13 test xanh** ([ghi chú](docs/lessons/project_data_registry.md)). Cùng ngày: [tài liệu bàn giao](docs/early/HANDOVER_ANTIGRAVITY.md) cho một công cụ AI khác.
- **29/06** chốt thiết kế M13 colony-sim thật ([quyết định](docs/lessons/project_m13_gameplay_design.md)).
- **04/07** **viết lại sinh thành phố** kiểu thành quách (trục chính, chia khối BSP, hẻm, nhà dãy) — [ghi chú](docs/lessons/project_citygen_wallcity.md).

## Giai đoạn 3 — Hệ Pawn/Job kiểu RimWorld (11/07 → 31/07) 🟡
- **11–16/07** [masterplan Pawn](docs/lessons/project_pawn_system_masterplan.md) (lệnh chỉ định, WorkGiver, lịch, thu hoạch, xây, khuân vác), [AI não ThinkTree/JobGiver](docs/lessons/project_pawn_ai_brain.md), [Job Pipeline (JobDriver/Toil)](docs/lessons/project_job_pipeline.md).
- **24–25/07** M18–M21 (kéo thả, trạng thái, nhàn rỗi, phản ứng đe doạ, trồng cây). Quy trình: *code → kiểm Play Mode → tôi xác nhận* (ngày xác nhận 14/08 được ghi).
- **28/07** ngày/đêm; đồng hồ toàn cục (test xanh); vòng đời tài nguyên. **30/07** M26 sản xuất, M27 thời khoá biểu, bảng token UI.

## Giai đoạn 4 — Thành phố sống & hiệu năng 2D (01/08 → 12/08) 🟢 (có ảnh từ 03/08)
- **01/08** 🟢 **City 2,8 → 92,6 FPS** nhờ tilemap chunk ([case 02](case-studies/02-draw-call-tilemap.md)). Công trình thành "nhà thật" (tường/phòng/cửa từng ô); bài học "xong = chơi được" ([case 14](case-studies/14-cach-lam-viec-voi-ai-va-bo-luat.md)).
- **02/08** 1000 NPC thành phố đi làm/về nhà; nội thất theo vai trò phòng + đặt sát tường (đo được 100% đồ lớn dựa tường).
- **03/08** [chốt nhịp thời gian](docs/lessons/project_time_pacing.md) (ngày = 18 phút thật, 30 TPS, lịch 4 tháng × 20 ngày). **Ảnh đầu tiên:** [`screenshot-20260803-194424.png`](images/timeline/screenshot-20260803-194424.png) — thành phố 2D nhìn từ trên xuống.
- **04/08** pawn người chơi điều khiển được ở City; `WildSceneInteraction` dùng chung cho Wild+City.
- **08/08** 🟢 **"Lag giờ đi làm": 4 lần đoán sai → đo → 713 ms → 3,26 ms** ([case 01](case-studies/01-lag-thanh-pho-bon-lan-doan-sai.md)).

## Giai đoạn 5 — Chiều sâu mô phỏng pawn (09/08 → 21/08) 🟡/🟢
- **09–10/08** combat thật trên Wild (M22), Companion (M29), quan hệ/độ thiện cảm (M29b).
- **13–15/08** sức khoẻ/chấn thương (M30), suy sụp tinh thần (M32), đau buồn (M35), xác chết (M36), ô trang bị theo dữ liệu (M34), gói nội dung CORE trung tính.
- **15/08** 🟢 **M37 — tách Core life-sim khỏi Xianxia** (199 file; phép thử "gỡ xianxia vẫn chạy") — [case 05](case-studies/05-tach-loi-khoi-tu-tien.md).
- **15–17/08** thú/dân có thân xác (M38, M44), sổ quan hệ (M41), tương tác pawn suy ra từ *quan hệ* (M42).
- **18–21/08** 🟢 màn hội thoại kiểu visual novel; đổi chác ⟂ mua bán; **hệ ký ức + lời thoại M48–M54: lặp lời 33,1% → 0%** — [case 04](case-studies/04-he-loi-thoai-khong-lap.md). Ảnh 18–21/08 (~42 tấm) trong `images/timeline`.

## Giai đoạn 6 — Gộp hệ thống, bộ luật, sự cố (24/08 → 29/08) 🟢
- **24/08** ảnh chụp UI/HUD/thế giới dày hơn — xem [bộ ảnh](images/timeline).
- **26/08** viết [CODE_SYSTEM_RULE](docs/CODE_SYSTEM_RULE.md) — mỗi luật gắn với một bug thật; lập catalogue tính năng.
- **27/08** 🟢 [sự cố HUD.prefab dùng chung → "hợp nhất có chủ đích"](case-studies/03-su-co-hud-prefab-hop-nhat.md); chủng tộc (M56), nhóm (M57), dân thành phố có thân xác.
- **28/08** pawn đi lại giữa nhiều ô world (M58–M60 giường), **lưu công trình theo ô (M61)** — [case 13](case-studies/13-do-tin-cay-luu-tru-va-su-co.md).
- **29/08** 🟢 Unity treo + RAM 11 GB → nguyên nhân & sửa (cùng case 13).

## Giai đoạn 7 — Pack Tensura, combat framework, sinh nhà, tách mod (03/09 → 05/10) 🟢 (git từ 12/09)
- **02–03/09** combat Mode Framework (M64): `ICombatDriver` cắm rời, RTwP ⟂ theo lượt, chọn lúc tạo ván ([M64](docs/lessons/project_m64_combat_mode_framework.md)). Ảnh trận theo lượt: [`screenshot-20260903-144045.png`](images/timeline/screenshot-20260903-144045.png). **06/09** pack Tensura (M66) — sau cất thành mod tắt (28/09).
- **07/09** 🟡 chốt chiến lược mod (JSON ⟂ hook C#, loại Lua) — [case 11](case-studies/11-mod-va-content-pack.md).
- **12–17/09** 🟢 **commit đầu tiên của git.** Migration `Vector2Int → Vector3Int` toàn engine; **sinh bố cục nhà nhiều tầng: nhiều lần hoàn tác, từ annealing sang BSP "0% ô thừa"** — [case 06](case-studies/06-thuat-toan-sinh-nha.md).
- **18–19/09** hệ biome + địa hình theo chunk; công trình hoang dã có ngân sách đảm bảo.
- **21–25/09** 🟢 **thế giới liền (seamless)**: cửa sổ sector, cư dân theo thành, spawn quái runtime; sửa khựng 430 ms qua ranh; sửa bản đồ đen; biên giới biome hữu cơ; vòng đời rừng.
- **26–27/09** 🟢 cư dân vĩnh viễn + đi giữa các thành; bắt thú trong trận; Field Codex; huấn luyện viên NPC; kế hoạch ngày theo dữ liệu; thám hiểm nhiều ngày.
- **28–30/09** 🟢 **tách nội dung ra mod** (`pokemon`, `xianxia`, `tensura`) — [case 11](case-studies/11-mod-va-content-pack.md). 29/09 avatar di chuyển liên tục (WASD); nạp WorldMap bằng Addressables.
- **01–05/10** 🟢 A* cấp ô của dân thành chạy thành **Burst job cho đường đi y hệt**; lõi noise sinh địa hình chạy Burst; bạn bè & hộ gia đình cho cư dân; **định dạng bản vẽ công trình chung + xưởng vẽ ngay trong game** (tôi chốt: "chỉ khi ta để bản thân vào người dùng, ta mới biết cái gì là tốt nhất").

## Giai đoạn 8 — Thế giới khối kiểu Minecraft (08/10 → 18/10) 🟢
- **08–12/10** prototype voxel → vẽ *game thật* bằng khối → chuột/POV/túi đồ, 2D và 3D là hai bộ vẽ của cùng dữ liệu. Ảnh: [`images/prototype`](images/prototype) (18 ảnh theo thứ tự).
- **13–15/10** chunk khối 16³ sửa được, cây/quặng/cây trồng thành khối, vật lý khối, ánh sáng, nước/dung nham chảy, khối có hình, texture sinh bằng code, **ánh sáng sang Texture3D + gộp mặt** — [case 07](case-studies/07-the-gioi-khoi-render-anh-sang-nuoc.md).
- **14–18/10** 🟢 **hiệu năng 3D: FPS → giật cục → vào game 12–15 s còn 3,8 s; GC 298 → 63 KB/khung; tìm đường không chặn** — [case 10](case-studies/10-hieu-nang-the-gioi-3d.md), [case 09](case-studies/09-tim-duong-canreach-findpath.md).
- **16–17/10** 🟢 **hệ công cụ/vật phẩm kiểu MC** (lực phá theo tag, `ItemUseDispatcher`, hotbar 9 ô = cầm, tay phụ, độ bền, ô trang bị theo gen/loài) — [kế hoạch](docs/lessons/project_tool_item_plan.md).

## Giai đoạn 9 — Viết lại bộ sinh thế giới & hoàn thiện (18/10 → 23/10) 🟢
- **18/10** mọi sinh vật có thân vật lý như avatar (bỏ luật "trong nhà không mô phỏng") + A* theo cỡ thân — [ghi chú](docs/lessons/project_creature_physics.md).
- **19–21/10** 🟢 **bộ sinh địa hình 3D liền**: spline, sông/hồ thật, điêu khắc, biome theo khối, hang nhiều loại; bug "cột đá vô tri" — [case 08](case-studies/08-sinh-dia-hinh-3d.md).
- **21/10** màn hình chính kiểu Minecraft; tốc độ đi theo khối; đo RAM bản build. **22/10** túi đồ 2D kiểu Minecraft; hotbar/lưới chế/tay phụ là ô thật.
- **23/10** 🟢 **trồng trọt: thiết kế tổng quát → 6 bước = 6 commit** — [case 12](case-studies/12-trong-trot-thiet-ke-roi-code.md).

---

## Mẫu hình lặp lại
1. **Thiết kế bằng văn bản trước, code sau.**
2. **Đo rồi mới sửa**, nhìn đuôi phân phối, ép tải.
3. **Báo cáo trung thực** (đã code / test xanh / verify Play Mode / chưa rõ nguyên nhân / nợ còn lại).
4. **Mỗi bug thành một luật** + test canh ranh giới.
5. **Lõi ⟂ nội dung, dữ liệu ⟂ hành vi.**
6. **Hoàn tác là bình thường** — thử, đo, lùi; lịch sử giữ lại bằng chứng.

> *Đính chính so với bản trước:* trận "lag giờ đi làm" 08/08 **không phải** do A* quét nhà dân (đó mới là 4 lần đoán đầu); thủ phạm thật, đo ở vòng 5, là `FindObjectsByType` trong vòng lặp lồng. Xem [case 01](case-studies/01-lag-thanh-pho-bon-lan-doan-sai.md).
