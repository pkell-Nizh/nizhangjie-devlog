# Case study 01 — "Lag giờ đi làm": 4 lần đoán sai, 1 lần đo đúng (713 ms → 3,26 ms)

**Thời gian:** 08/08/2026 · **Phạm vi:** thành phố ~874 dân (C#/Unity) · **Kết quả:** khung hình chính **713 ms → 3,26 ms** (~219 lần), 0 khung > 100 ms.

> **TL;DR (EN):** The city simulation lagged badly at "commute hour". We spent four rounds optimising pathfinding based on reading code — each helped a little, none found the real cause. When I finally profiled, 97% of the AI cost was a different system (a full-scene `FindObjectsByType` inside a nested loop). Lesson: *a user's symptom is data, not a diagnosis — measure first.*

## Bối cảnh
Thành phố có ~874 dân. Đến 7–10 giờ trong game (giờ NPC đi làm) game giật nặng. Người chơi mô tả: "giật kiểu lạ, như frame chỉ nhích khi một pathfinding vừa sinh ra". Triệu chứng nghe **rất giống** pathfinding.

## Hành trình (ghi lại cả những bước sai — đó là phần có giá trị nhất)

### Vòng 1 — Đọc code, thấy A* quét cả nhà dân
- Nguyên nhân đoán: BFS "mù" kiểm tra tới-được chạy trước A*, duyệt cả nội thất nhà người lạ vì sàn nhà có cost bằng đường phố.
- Sửa: cờ `avoidPrivateInterior` (nội thất chỉ được vào nếu là *đích*), cộng thêm thiên vị "chọn chỗ làm gần nhà" (`workLocalityWeight`).
- Thêm 4 test hồi quy (dựng cố ý một tình huống đường thẳng xuyên nhà để chứng minh baseline thật sự có vấn đề).

### Vòng 2 — Nghiên cứu kỹ thuật ngành
- Đọc về flow field và path caching trong city builder; đối chiếu RimWorld (chia region 12×12, project đã làm giống).
- Nhận ra: đường đi làm/về nhà của một người **không đổi** (layout đóng băng) nhưng mỗi ngày A* tính lại y hệt cho mọi NPC → thêm cache theo cặp cửa.
- **Tự bắt một bẫy trước khi code sai:** tường/cửa thành phố là vật thể xây được, có thể bị phá lúc chơi → cache phải vô hiệu hoá theo `Regions.Version`, không phải theo toạ độ.

### Vòng 3 — Chẩn đoán lại bằng cách đọc chính code vừa sửa
- Phát hiện nhánh fallback chạy **A\* không trần trên 160.000 ô** — một lần gọi treo cả khung, và `avoidPrivateInterior` còn đẩy thêm chuyến vào nhánh đó.
- Sửa 3a: trần cứng 6000 nút. **Tự bắt bug phát sinh:** kết quả null do bị cắt ngang bị cache hiểu là "vĩnh viễn không có đường" → sẽ nhốt NPC trong nhà mãi. Sửa: chỉ cache đường *thành công*.
- Sửa 3b (ý của tôi, mạnh hơn mọi sửa trước): thu hẹp *đồ thị* thay vì lọc sau — dân thường chỉ đi trên ô Road nên dựng một đồ thị road-only thứ hai: **2369 → 590 region (nhỏ hơn ~4 lần)**.
- Kết quả tôi báo: "bớt lag hơn, nhưng vẫn lag."

### Vòng 4 — Đa luồng pathfinding
- Tôi hỏi Claude "mình có đang multithread pathfinding không?" → không, và đó là quy ước cũ có chủ đích. Claude đề xuất đo Profiler trước; tôi vẫn chọn làm đa luồng luôn.
- Thiết kế chia hai pha để tránh data race: `PrepareQuery` (chỉ main thread, làm mọi thao tác **ghi** như dựng lại region, ghi cache) và `ExecutePreparedQuery` (luồng nào cũng được, chỉ **đọc**).

### Vòng 5 — Cuối cùng cũng đo Profiler
Kết quả **lật ngược** 4 vòng trước:

| (CityScene, 874 dân, tua 8×) | Trước | Sau |
|---|---|---|
| Main Thread trung bình | **713 ms** (~1,4 FPS) | **3,26 ms** |
| `BehaviourUpdate` | 663 ms | 0,52 ms |
| `JobGiver_StandingOrder` / lần gọi | **77,07 ms** (97% chi phí AI) | 0,123 ms |
| Khung > 100 ms | 25/25 | **0** |

**Thủ phạm thật:** `IdleActivityPipeline.ResolveNearbyFurniture` — không liên quan pathfinding. Ba lỗi cộng dồn:
1. `WildPlayerToken.At()` gọi `FindObjectsByType` (quét toàn scene) **mỗi lần**, và nó nằm trong vòng lặp lồng (mọi món nội thất × mọi ô kề). → thay bằng sổ token tĩnh tự đăng ký.
2. Lọc theo khoảng cách chạy **sau** phần đắt nhất. → lọc thô trước, với biên an toàn chứng minh được không loại nhầm.
3. `PawnAiBrain.HandleTick` cũng quét scene mỗi tick.

**Vì sao chỉ City mới chết:** code gốc viết cho map Wild (vài chục công trình) và ghi rõ trong comment "quét trực tiếp, pawn ít nên chấp nhận được". Khi đưa hệ AI sang City, có **6906 món nội thất** → giả định "ít" sai ~100×.

## Điều học được
- **Triệu chứng tôi mô tả là dữ liệu, không phải chẩn đoán.** Bốn vòng tối ưu "nghe hợp lý" đều là cải thiện thật (tôi xác nhận bớt lag) nhưng sai hệ.
- **Nhìn đuôi phân phối** (p99, max, số khung vượt ngưỡng) — trung bình 2,68 ms vẫn có 20 khung > 100 ms.
- **Ép tải mới lộ lỗi:** ở tốc độ 1× game 373 FPS, tua 8× chỉ còn 1,4 FPS.
- **Khi đưa hệ sang quy mô lớn hơn ~100×, rà lại mọi chỗ ghi "chấp nhận được vì ít".**
- **Bẫy của chính phép đo:** MCP biên dịch C# trên main thread (200 ms–2,3 s) → nếu bộ lấy mẫu còn chạy lúc đó, phép đo tự tạo ra đỉnh nó báo cáo (đã dính: max 1198 ms trong khi p95 chỉ 3,96 ms). Unity cũng không tick khi cửa sổ mất focus (`runInBackground`).
- **Biết dừng:** đo lại 09/10 (1000 dân): `CityNpc.SlowTick` 3,9 ms/lần, p95 5,4. Tôi quyết định *không* tối ưu thêm vì chưa phải nợ kỹ thuật — ghi lại để không ai đề xuất lại vô ích.

## Bằng chứng
- Ghi chú gốc (đủ 5 vòng + đo lại): [`docs/lessons/project_city_commute_perf.md`](../docs/lessons/project_city_commute_perf.md)
- Luật rút ra trong bộ luật dự án: *C3 "Đo trước khi tối ưu"*, *B8 "Cấm quét toàn cục trong vòng lặp"* → [`docs/CODE_SYSTEM_RULE.md`](../docs/CODE_SYSTEM_RULE.md)
