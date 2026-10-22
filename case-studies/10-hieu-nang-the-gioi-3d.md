# Case study 10 — Hiệu năng thế giới 3D: từ "FPS trung bình ổn" đến "hết giật cục" (14–18/10/2026)

**Kỹ năng:** Unity Profiler (ProfilerRecorder, hierarchy view), phân tích đuôi phân phối, GC, luồng nền, ngân sách thời gian theo khung, quy trình đo có kiểm soát.

> Mọi số liệu dưới đây lấy từ ghi chú đo lúc làm; điều kiện (Editor, cửa sổ 1100×500, thành phố, 9–25 ô world) ghi trong tài liệu gốc.

## Bước 1 — Phân rã khung hình (14/10)
WildLocal 9 ô world có thành (6257 mảnh tường, 1690 công trình): khung ~20 ms với Profiler bật. Thủ phạm *script* là **`ContextActionController.Update` 4,1 ms** (quét ô trước mặt mỗi khung); phần còn lại chờ GPU (5,2 ms) và các hệ nhỏ. Tick **không** phải thủ phạm (1× ⟂ 8× chỉ lệch ~1 ms).
- Sửa: chỉ quét khi ô nhắm đổi / mỗi 0,15 s / khi bấm phím → **4,1 → 0,5 ms**; chỉ mục tường theo ô (đối chiếu 2086 ô: 0 lệch).
- Kết quả trung thực: 112 → 118,6 FPS, p50 8,0 → 6,2 ms, **nhưng p95 xấu đi 13,3 → 18,4 ms** — ghi "chưa rõ nguyên nhân" thay vì tuyên bố thành công.

## Bước 2 — Đổi chỉ số: đo *giật cục*, không đo FPS (15/10)
FPS trung bình đã ổn (~124) nhưng người chơi vẫn thấy khựng. Đo giật trên 5×5 ô: 
- Đập 1 khối: 5–13 ms (dựng lại lưới chunk *ngay trong sự kiện*). Chuỗi đổi ô (nước chảy, cát rơi) = mỗi ô dựng lại một lần → nên gom "chunk bẩn".
- Qua ranh ô world: ~1 s khựng (ghép ô 9–56 ms/lượt; sau đó 4 khung 180–340 ms do quái 65–118 ms, đường NPC thành 42–117 ms).
- Giật định kỳ khi đứng yên: `DoorTrafficSystem` 10–25 ms, `BeastBehaviour` 8–23 ms vài giây một lần.
- `JobGiver_HaulToStorage`: đồ rơi không dùng được → quét cả 686 kho, mỗi pawn mỗi nhịp 3–6 ms.

## Bước 3 — Hạ tầng phân rã tick (`TickSpread` + `BudgetedQueue`)
- Pha tác nhân = FNV-1a(khoá) mod chu kỳ, chạy theo **tick** của `GameClock` (tất định, pause là đứng); khoá là id dữ liệu, **không** `GetInstanceID`/`string.GetHashCode`.
- `BudgetedQueue`: khoá trùng thì việc mới nhất thắng; `Drain(budgetMs)` FIFO. Test bắt được bug ở bản đầu (Remove rồi xếp lại phải xuống *cuối*).
- Kết quả đứng yên 40 s: `PawnAiBrain` max **23–43 → 0,22 ms/tick**; 8 lần đập khối cùng chunk → 1 lần dựng, ~1 ms/lần (trước 5–13). Test 18/18.
- Dựng region *dần*: đo thật 80 ms + 41 ms mỗi ô world → `DeferBulk` ngân sách 2 ms/bộ/khung.

## Bước 4 — Chia sim ⟂ render: **đo trước rồi mới quyết** (17/10)
Câu hỏi: "có nên tách game loop khỏi render loop?" Đo: sim ~1,5 ms/khung, **render thread 4,4 ms**; giật còn lại là A* nhịp chậm của dân thành (~25 ms). → *Kết luận:* đẩy cả sim ra luồng riêng gần như không lợi; làm từng phần (đẩy phần nặng ra luồng nền) + đặt luật **B10** "sim là dữ liệu C# thuần, tick không đụng Unity" kèm test canh (`Sim Purity`, mốc chỉ được hạ): dọn 24 → 6 chỗ dùng API Unity trong code sim.

## Bước 5 — Ô 240, nạp/gỡ theo chunk, cắt khựng qua ranh (16/10)
Mô hình lai: tạo theo ô world ở luồng nền, đưa vào/ra game theo chunk, ngân sách ms/khung.
| Qua ranh ô | trước | sau |
|---|---|---|
| ApplyBuilt | 85 ms | 1,2 |
| Destroy | 101 | 2 |
| VW.Apply | 192 | ≤ 8 |
| Beast OnTick | 166 | 29 |
| Khung tệ nhất | ~190 | ~100 (p95 ~30 → ~11 sau đợt ô có thành) |
Cùng đợt: BFS tới-được hai chiều (27 → 8 ms), `RecomputeBlock` (12–14 → 4 ms).

## Bước 6 — Vào game 12–15 s → 3,8 s (17/10)
Đo bằng `LoadTrace`: 12–15 s = dựng đồng bộ 25 ô (5–6 s) + một bước tô phòng 4,3 s (lặp flood fill ~400 ô × mỗi viên tường × 4 hướng). Sửa: chỉ ô dưới chân dựng đồng bộ, 24 ô còn lại nạp nền; tô phòng nhớ vùng hở; vòng vẽ *tròn* theo chunk. **→ 3,8 s**, 25/25 ô đủ sau vài giây. Số RAM: ~28 MB/ô (đo sau GC), ~1,55 GB cố định → kết luận *không đáng viết lại lõi* để nạp theo chunk nếu chỉ vì RAM.

## Bước 7 — Bay 3D / GC (18/10)
- PerfPanel trong game (F8) để người chơi vừa chơi vừa đo, báo cáo ra file.
- GC allocation khi bay: **298 → 134 → 87 → 63 KB/khung**, `GC.Collect` 141 → 76 → 0. Bẫy: closure bắt tham số tạo rác; quét `ContextAction` 66 → 2 KB/lần (`foreach` trên `Footprint`).
- Dựng lưới hang + dựng lại theo ánh sáng chuyển sang **luồng nền**; `VW.Want` max 17,8 → 11,9 ms.
- Một nghi vấn còn lại ghi trung thực: "~5 ms chưa giải thích".

## Quy trình đo (đáng giá hơn từng con số)
- Nhìn **p95/p99/max/số khung vượt ngưỡng**, không nhìn trung bình.
- `Application.runInBackground = true` *sau* domain reload; `profileEditor = false`; xoá khung → bật → ForceCell trong cùng lệnh; đọc sau ~5 s (bộ đệm 2000 khung trôi nhanh).
- Remote desktop của tôi làm chờ GPU 5–33 ms/khung → **không so FPS tuyệt đối với số cũ**.
- Lần đầu một ô có thành thường là JIT vài ms — đừng tin số chạy lạnh.
- Mọi thay đổi để lại **ProfilerMarker** rẻ (`VW.*`, `SW.*`, `VGV.*`, `PRG.*`, `RP.*`, `Beast.*`, `Clock.<Lớp>.<Hàm>`).

## Bằng chứng
[fps_profile](../docs/lessons/project_fps_profile_2026-10-14.md) · [hitch_profile](../docs/lessons/project_hitch_profile_2026-10-15.md) · [tick_spread](../docs/lessons/project_tick_spread.md) · [walk_hitch](../docs/lessons/project_walk_hitch_2026-10-17.md)
