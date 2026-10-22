---
name: project-city-commute-perf
description: "Lag giờ đi làm City + A* quét nhà dân — root cause + fix avoidPrivateInterior + job-locality bias, ĐÃ CODE+test Editor xanh 2026-08-08, CHƯA verify Play Mode"
metadata: 
  node_type: memory
  type: project
  originSessionId: 0cd97dfe-d9e3-4f06-8daa-26026bf6ce16
  modified: 2026-08-08T07:52:34.627Z
---

**User báo 2026-08-08:** test giờ 7-10h (giờ NPC đi làm) cực lag, và phát hiện A* quét cả nhà dân không liên quan.

**Root cause xác nhận bằng code, không phải đoán:**
- `TileType.Floor` (Cell.cs) chỉ dùng cho SÀN/CỬA nội thất nhà City (3 call site, đều ở `CityGenerator.StampToGrid`) — không dùng ở đâu khác, kể cả Wild. Vì thế dùng thẳng `Type == Floor` làm cờ "nội thất riêng tư", KHÔNG cần thêm field mới trên `Cell`.
- Floor cost=1.0 (fallback `LocalPathfinder.GetMoveCost`, không có entry riêng trong `terrain_tiles.json`) — ngang Grass, KHÔNG bị chặn. Mọi nhà đều có cửa mở thẳng ra ô đường (Entrance) không qua tường, nên A*/flood-fill coi nội thất là không gian hợp lệ y hệt đường phố.
- Chỗ đau thật: `RegionReachability.BfsRegions` ([RegionReachability.cs:69](Assets/Scripts/LocalMap/Pathfinding/RegionReachability.cs:69)) là BFS MÙ (không heuristic) chạy TRƯỚC A* cho MỌI chuyến (`CanReach`, nấc 1 filter rẻ). Vì cả phố + mọi nội thất nhà dân dính chung 1 thành phần liên thông, lần đầu hỏi 1 cặp (from,to) mới (rất phổ biến giờ đổi ca vì 155+ người/giờ mỗi người 1 tuyến khác nhau) phải duyệt qua các "ngóc ngách nhà người lạ" trước khi tới đích — đúng như user quan sát. Layer CellAStar cũng bị: doorway nhà nào cũng là node có cost gần-bằng-tối-ưu nên lọt vào open set, tốn thời gian duyệt+loại dù cuối cùng không dùng.
- **Đã có SẴN 3 cơ chế giảm tải khác** ([CityNpcSystem.cs:608-629](Assets/Scripts/World/City/Npc/CityNpcSystem.cs:608)) — `_pathBudgetMs`/`_spreadDeparture`/`_abstractTripCells` — nhưng chỉ giảm SỐ LƯỢNG truy vấn/frame, không giảm CHI PHÍ mỗi truy vấn. Đây là lý do dù có 3 cơ chế đó vẫn lag.

**Fix đã code (KHÔNG đổi hành vi Wild — Wild không dùng TileType.Floor, và param mới default false):**
- `PathGrid._isInterior` (bool[], build từ `cell.Type == TileType.Floor` trong `Recompute()`) + `IsInteriorAt(x,y)` — [PathGrid.cs](Assets/Scripts/LocalMap/Pathfinding/PathGrid.cs). KHÔNG đụng `EnterCost`/`WalkableAt` — vẫn đi được bình thường, chỉ là cờ lọc thêm.
- `RegionPathfinder.FindPath(..., bool avoidPrivateInterior = false)` → `CellAStar` skip neighbor nếu `avoidPrivateInterior && IsInteriorAt(n) && n != to` — **trừ khi chính là đích** (destination-exemption, quan trọng: user chốt phải giữ khả năng đi bộ THẬT vào nhà, vd bấm Move vào 1 căn cụ thể — path tới đúng căn đó vẫn hoạt động, chỉ KHÔNG dùng nhà khác làm lối tắt dọc đường).
- `CityNpcSystem.DrainPathQueue` gọi `pf.FindPath(trip.From, trip.To, avoidPrivateInterior: true)` — chỉ City NPC commute bật cờ này. Chỗ khác gọi `RegionPathfinder`/Wild giữ nguyên mặc định false.
- Test mới `RegionPathfinderTest` Test 9 (4 case) — dựng scenario đường thẳng ngắn nhất CHỦ Ý xuyên "nhà" (cùng cost nên A* tự nhiên đi xuyên, chứng minh baseline có vấn đề thật chứ không phải giả định), rồi so sánh cost thật (không phải path.Count — bước chéo gộp 2 trục nên số bước có thể hoà dù path dài hơn) khi bật cờ.

**Ý thứ 2 của user, cũng đã code — chọn việc GẦN NHÀ (giảm luôn số/độ dài chuyến, bổ trợ fix trên chứ không thay thế):**
- `NpcGenConfig.workLocalityWeight` (0..1, mặc định 0.6) + `workLocalityHalfDistanceCells` (mặc định 60) — 2 field JSON mới, đã sync cả `Assets/Data` lẫn `StreamingAssets/Data` ([[feedback_data_folder_split]]).
- `NpcGenerator.ApplyWorkLocalityBias` — nhân thêm hệ số `Lerp(1, 0.5^(dist/half), workLocalityWeight)` vào trọng số diện tích cũ (`BuildWorkWeights`), tính theo Manhattan distance giữa `Entrance` nhà và `Entrance` chỗ làm. `workLocalityWeight=0` → y hệt hành vi cũ (chỉ diện tích).
- Áp dụng ở CẢ `NpcGenerator.Generate` (sinh lần đầu) LẪN `NpcLifeOps.DailyTurnover` (bù dân) — tiện thể fix luôn 1 bất nhất có sẵn: `DailyTurnover` trước đây bốc việc THUẦN ĐỀU (`works[rng.Next(...)]`), không hề theo diện tích như lúc sinh lần đầu.

**Test Editor xanh 2026-08-08:** `RegionPathfinderTest` 21/21 (gồm 4 case mới) · `MNpcTest` 29/32 (3 fail PRE-EXISTING không liên quan: role skill map thiếu "alchemy"/"bank" trong `roleSkills` JSON, castle family ở nhà role≠residence, tỉ lệ dạo phố đỉnh điểm lệch tuning — cả 3 đã tồn tại trước session này, không phải do fix trên) · `MNpcBrainTest` 38/39 (1 fail pre-existing về sleep-interrupt, không liên quan).

**CHƯA verify Play Mode** — theo [[feedback_done_means_playable]], code xanh KHÔNG phải "xong". Cần: vào CityScene, tua nhanh tới giờ đổi ca (7-10h), quan sát FPS/khựng hình có cải thiện thật không; xác nhận player vẫn Move được vào 1 căn nhà cụ thể (destination-exemption); xác nhận NPC có xu hướng làm gần nhà hơn quan sát được (khó đo bằng mắt, có thể thêm debug log tạm nếu cần).

---

## Vòng 2 (2026-08-08, cùng ngày) — user báo VẪN cực lag sau vòng 1, yêu cầu tìm kỹ thuật ngành

**Research (WebSearch, có nguồn):** 2 kỹ thuật lớn cho crowd pathfinding hàng nghìn agent —
1. **Flow field** — 1 lần Dijkstra ngược từ ĐÍCH, mọi agent đọc hướng O(1)/frame. Ăn tiền nhất khi NHIỀU agent cùng hội tụ về ÍT đích (đúng dạng "giờ đi làm": nhiều người cùng đổ về vài chỗ làm/về vài khu nhà).
2. **Path caching theo target + version cost-field** — nếu cost-field/đích không đổi thì khỏi tính lại. RimWorld xác nhận: chia region 12×12 (project ĐÃ mirror đúng cái này ở PathRegionGrid), "Region Caching System" giảm 4 lần chi phí bằng cách pawn chỉ đăng ký lại region khi ĐỔI region chứ không mỗi tick.

**Nhận ra sau research: City đã ở tình huống LÝ TƯỞNG cho path caching mà TRƯỚC ĐÓ CHƯA làm.** Layout City ĐÓNG BĂNG sau khi sinh (CLAUDE.md §5), Entrance mỗi nhà cố định — nên đường (nhà)↔(chỗ làm) của MỘT NGƯỜI không bao giờ đổi, nhưng `HandleHourTick` mỗi NGÀY lại bắt A* tính LẠI Y HỆT cho MỌI NPC còn sống (turnover chỉ ~0.4%/ngày ⇒ gần như toàn bộ dân số trả tiền A* mỗi ngày cho một câu trả lời KHÔNG ĐỔI). Đây là chỗ ăn tiền lớn hơn hẳn avoidPrivateInterior của vòng 1.

**Đã code:** `CityNpcSystem._commuteCache` (Dictionary theo cặp doorway, CHỈ cache trip AtHome/AtWork — Wandering/Visiting đích ngẫu nhiên mỗi lần nên cache vô ích, chỉ phình bộ nhớ, cố tình KHÔNG cache). Cache value NULL (chuyến bất khả thi) cũng được nhớ — khỏi thử lại A* mỗi ngày cho cặp không có đường. Polish "đẩy ra giữa phố" giờ chạy ĐÚNG 1 LẦN lúc cache-miss (không phải mỗi NPC) — nhiều NPC cùng tuyến CHIA SẺ chung 1 List instance, an toàn vì chỉ đọc (`PathIndex` là field riêng từng NPC, `Step()` không bao giờ ghi vào `Path[i]`).

**⚠ Bẫy đã tự bắt trước khi code sai:** City tường/cửa giờ là `BuildPiece` THẬT, deconstruct được lúc chơi ([[project_city_walls_furniture]]) — nghĩa là grid CÓ THỂ đổi giữa lúc chơi, không chỉ lúc sinh layout. Cache theo toạ độ thuần sẽ cũ khi ai đó phá tường mở lối tắt mới hoặc chặn lối cũ. Fix: mirror ĐÚNG cách `RegionReachability` tự vô hiệu — so `pf.Regions.Version` mỗi lần `DrainPathQueue` chạy, khác thì vứt sạch cache (rẻ hơn dò từng entry). `HandleLayoutReady` (regen toàn bộ) reset cả `_commuteCacheVersion = -1` vì instance `RegionPathfinder` MỚI có `Version` riêng bắt đầu lại, có thể TRÙNG SỐ với bản cũ.

**Compile sạch, chưa thêm test Editor riêng cho cache (khó test nếu không có Play Mode — bản chất là tối ưu hiệu năng, không đổi kết quả path).** CHƯA verify Play Mode thật (FPS trước/sau lúc đổi ca) — cần user tự cảm nhận hoặc mượn Unity Profiler (`mcp__UnityMCP__manage_profiler`) chụp lúc 7-10h nếu vẫn lag để tìm đúng hot path thay vì đoán tiếp. Nếu cache này vẫn chưa đủ, bước leo thang tiếp theo là **flow field cho các chỗ làm/khu nhà đông người** (kỹ thuật #1 ở trên) — phức tạp hơn nhiều, chỉ nên làm khi có số đo thật chỉ đích danh pathfinding vẫn là nút thắt.

Sources: [Pathfinding in city builders — GameDev.net](https://www.gamedev.net/forums/topic/691128-pathfinding-in-city-builders-store-the-paths/) · [Flow Fields — moonjump.com](https://moonjump.com/game-dev-mechanics-flow-fields-how-it-works/) · [AI Paths Unpacked: RimWorld Routing — rimworldhub.com](https://rimworldhub.com/post/ai_paths_unpacked_rimworld_routing_quirks__tactics)

---

## Vòng 3 (2026-08-08, cùng ngày) — user báo "giật kiểu lạ, như frame chỉ nhích khi 1 pathfinding vừa sinh ra"

**Chẩn đúng nguyên nhân bằng cách đọc lại chính code vừa sửa:** [RegionPathfinder.cs:98](Assets/Scripts/LocalMap/Pathfinding/RegionPathfinder.cs:98) — khi tầng corridor thất bại, rơi thẳng về **A\* KHÔNG TRẦN trên TOÀN BỘ 160.000 ô**. `avoidPrivateInterior` (vòng 1) khả năng làm corridor thất bại NHIỀU HƠN (loại luôn ô nội thất mà corridor hẹp từng dựa vào), đẩy nhiều chuyến hơn vào ca fallback đắt đỏ này — 1 lần gọi treo cả frame, không có framerate cố định vì độ dài treo phụ thuộc chính truy vấn đó tốn bao lâu.

**Fix 3a — trần cứng:** `RegionPathfinder.MaxCellExpansions = 6000` (node A* cấp ô) — vượt trần thì `CellSearchAbortedCount++` rồi trả null, KHÔNG throw. **Bẫy tự bắt kèm theo:** null-do-cắt-ngang bị `_commuteCache` (vòng 2) hiểu nhầm là "vĩnh viễn không có đường" → NHỐT NPC trong nhà mãi mãi. Sửa: cache CHỈ lưu path THÀNH CÔNG, không lưu null.

**Fix 3b — ý user đề xuất, đúng và mạnh hơn hẳn mọi fix trước: thu hẹp GRAPH tận gốc thay vì lọc SAU khi build xong.** NPC dân thường CHỈ BAO GIỜ đi trên `TileType.Road` (Entrance mọi building LÀ ô đường; người trong nhà không hề pathfind từng ô, vị trí đặt thẳng). Vậy mà graph tìm đường trước giờ vẫn gồm HẾT mọi loại ô đi được (đường+sân+nội thất+đất trống) dù NPC không bao giờ cần 84% phần còn lại đó.

- `CitySceneBootstrap.RoadPathfinder` (MỚI, song song `Pathfinder` cũ) — 1 `RegionPathfinder` THỨ HAI dựng trên CÙNG `GridSystem`, nhưng cost table riêng (`BuildRoadOnlyTiles`): chỉ Road walkable (giữ nguyên cost/speed thật từ JSON), MỌI type khác `passable=false` — duyệt hết `enum TileType` chứ không liệt kê tay, để type mới sau này tự động bị chặn thay vì lọt lưới.
- `CityNpcSystem.DrainPathQueue` dùng `RoadPathfinder` làm graph CHÍNH; chỉ lùi về `Pathfinder` (đủ mọi type) cho ca HIẾM — building "khối đặc" không Entrance (`NpcGenerator.ComputeHomeCapacity` ghi nhận loại này tồn tại, có thể vẫn được gán làm CHỖ LÀM dù không phải nhà ở), `DoorwayOf()` khi đó trả về 1 ô TRONG nhà (`StableSpotIn`) — Blocked tuyệt đối trên graph road-only nên tự rơi xuống graph đủ (check `roadPf.PathGrid.WalkableAt(From) && WalkableAt(To)`).
- **Số đo thật (probe qua `execute_code`, map 400×400 mặc định, seed 777):** graph đủ = **2369 region**; graph road-only = **590 region** — giảm ~4 LẦN kích thước graph mà mọi tầng phía trên (BFS reachability, corridor A*, cell A* worst-case) đều ăn theo.
- `avoidPrivateInterior` (vòng 1) giờ THỪA cho nhánh road-only (nội thất vốn không có trong graph đó) nhưng vẫn giữ `true` cho cả 2 nhánh — vô hại, đơn giản hoá code, và vẫn cần thật cho nhánh fallback hiếm.

**Compile sạch, `RegionPathfinder Tests` 21/21, `Npc Tests` 29/32 (đúng 3 fail cũ, không liên quan, không đổi).** Lưu ý: `MNpcTest`/`RegionPathfinderTest` KHÔNG dựng `CitySceneBootstrap` (MonoBehaviour) nên không tự động test được chính `RoadPathfinder` — **CHƯA có test Editor riêng cho graph road-only + đường vòng fallback building-khối-đặc**, và tất nhiên vẫn **CHƯA verify Play Mode**.

**Kết quả user báo sau vòng 3: "bớt lag hơn, nhưng vẫn lag."**

---

## Vòng 4 (2026-08-08) — MULTITHREAD pathfinding

User hỏi "mình có đang multi threads pathfinding không" — **KHÔNG**, và đó là quy ước cũ có chủ đích (CLAUDE.md §6: "Job System/Burst/Collections — chưa dùng — Sim single-thread"). Mọi A* chạy đồng bộ ngay trong `Update()`. Mình đề xuất đo Profiler trước để tránh đoán mò vòng 4, **user chốt làm multithread luôn** — làm theo.

**LUẬT CHIA VIỆC (quan trọng nhất, đừng phá):** gần như MỌI thứ trong hệ pathfinding GHI state khi bị hỏi — `Regions.EnsureRebuilt()` (re-flood section dirty), `Reachability.CanReach` (ghi `_cache`), `RegionCost()` (ghi `_regionCost`). Gọi thẳng `FindPath` từ thread nền = data race trên Dictionary dùng chung. Nên tách 2 PHA:
- **`RegionPathfinder.PrepareQuery` — CHỈ MAIN THREAD**, làm hết phần GHI: `EnsureRebuilt` tường minh + `CanReach` + `BuildCorridor`. Rẻ (tra graph + cache).
- **`RegionPathfinder.ExecutePreparedQuery` — THREAD NÀO CŨNG ĐƯỢC**, chỉ còn A* cấp ô thuần ĐỌC mảng. Đây mới là phần tốn CPU thật.
- Hạ tầng phụ trợ: `PathRegionGrid.RegionAtNoRebuild` (đọc thuần, KHÔNG kích hoạt re-flood) + `HasDirtySections`; `CellAStar(..., bool threadSafe)` chọn `RegionAtNoRebuild` thay `RegionAt`; `CellSearchAbortedCount` chuyển sang `Interlocked.Increment`/`Volatile.Read`.

**`CityNpcSystem`:** cache hit → dùng ngay (đồng bộ, khỏi thread); cache miss → `PrepareQuery` (main) rồi `Task.Run(ExecutePreparedQuery)`, kết quả vào `ConcurrentQueue`, `DrainCompletedPaths()` (main thread, gọi đầu `Update`) mới `SendAlong`/đụng state game. Trần `_maxInFlight = 8` (thả vô hạn thì ThreadPool tranh CPU với chính main thread). Polish "đẩy ra giữa phố" chuyển sang chạy ở main thread lúc nhận kết quả (nó đọc `_bootstrap.Grid`). Tách `ApplyPath()` dùng chung cho cả 2 đường vào — 2 bản logic là chắc chắn lệch.

**3 bẫy đồng thời đã bịt (đều IM LẶNG nếu bỏ sót):**
1. **Id NPC TRÙNG giữa 2 lần sinh quần thể** (`npc:0`, `npc:1`… đánh lại từ đầu) ⇒ `Registry.Get(id)` KHÔNG phát hiện được job của thế hệ cũ, nó trả về một người HOÀN TOÀN KHÁC và gán cho họ đường của người đã biến mất. Fix: `_generation` tăng ở `HandleLayoutReady`+`OnDisable`, job mang theo `Generation`, lệch thì vứt. (Task không huỷ giữa chừng được — chúng vẫn chạy nốt rồi bị loại lúc về.)
2. **`_inFlight` âm** — reset `=0` lúc sinh lại nhưng job cũ về vẫn `--` ⇒ âm thì luôn `< _maxInFlight`, trần mất tác dụng. Fix: `Mathf.Max(0, _inFlight - 1)`.
3. **Grid đổi giữa lúc job chạy** (phá tường) ⇒ đường vừa tính có thể xuyên chỗ giờ đã chặn. Fix: so `Regions.Version` lúc nhận kết quả, khác thì vứt.
Cộng thêm: `Debug.LogError` cho exception trong Task đặt Ở MAIN THREAD lúc drain (Debug.Log không an toàn ngoài main thread), nuốt im lặng sẽ thành "NPC tự nhiên không đi làm nữa".

**Verify bằng probe `execute_code` (không chỉ compile):** ① 60/60 cặp doorway thật → `PrepareQuery`+`ExecutePreparedQuery` cho path **giống hệt** `FindPath` đồng bộ; ② `Parallel.For` 120 truy vấn chạy SONG SONG THẬT → **0 exception, 0 lệch kết quả**. Test cũ: `RegionPathfinder` 21/21, `Npc` 29/32 (3 fail cũ không đổi).

**VẪN CHƯA verify Play Mode.** Nếu sau vòng này vẫn giật, gần như chắc chắn nút thắt KHÔNG còn ở pathfinding (4 vòng đã dồn hết: thu hẹp graph 4×, cache tuyến, trần node, đẩy sang thread nền) — bước tiếp theo BẮT BUỘC là đo Profiler thật (`mcp__UnityMCP__manage_profiler`) thay vì đoán.

---

## Vòng 5 (2026-08-08) — ĐO PROFILER: THỦ PHẠM THẬT KHÔNG PHẢI PATHFINDING

**User chốt "đo profiler đi" — và kết quả lật ngược 4 vòng trước.** Bài học lớn nhất của cả chuỗi này: *4 vòng tối ưu dựa trên đọc-code + suy luận đã tối ưu NHẦM HỆ.* Chúng vẫn có ích (user xác nhận "bớt lag hơn") nhưng chi phí ÁP ĐẢO nằm ở chỗ khác hoàn toàn.

**Số đo (CityScene, 874 dân, tua 8x — tua nhanh mới lộ, ở 1x mọi thứ trông ổn 373 FPS):**
| | Trước | Sau khi sửa |
|---|---|---|
| Main Thread avg | **713 ms/frame** (~1.4 FPS) | **3.26 ms** |
| BehaviourUpdate avg | 663 ms | 0.52 ms |
| `JobGiver_StandingOrder` avg | **77.07 ms/lần gọi** (97% tổng chi phí AI) | 0.123 ms |
| Frame >100ms | 25/25 | **0** |

**THỦ PHẠM: `IdleActivityPipeline.ResolveNearbyFurniture` (M19 Idle, Tier 4) — KHÔNG phải pathfinding.** Ba lỗi cộng dồn, chỉ lộ ở quy mô City:
1. **`WildPlayerToken.At()` gọi `FindObjectsByType` MỖI LẦN** — mà nó bị gọi trong vòng lặp LỒNG (mọi món nội thất × mọi ô kề footprint). `FindObjectsByType` quét toàn scene; **không bao giờ được nằm trong vòng lặp**. Sửa: sổ token tĩnh `WildPlayerToken.All` (OnEnable/OnDisable tự ghi danh) — hành vi giữ nguyên vì `FindObjectsByType` mặc định cũng bỏ qua object inactive.
2. **Lọc khoảng cách chạy SAU phần đắt** — `FindStandCell` (quét footprint × 4 hướng, mỗi ô tra token) chạy cho MỌI món khớp trên TOÀN BẢN ĐỒ, rồi mới vứt theo `dist > def.radius`. Sửa: lọc thô bằng `wb.Origin` với biên `radius + 1` TRƯỚC (ô đứng luôn kề footprint nên chênh tối đa 1 ⇒ không loại nhầm).
3. `PawnAiBrain.HandleTick` cũng `FindObjectsByType<WildPlayerToken>` mỗi tick → dùng sổ luôn (chụp ra list tạm vì job có thể spawn/huỷ token giữa vòng lặp).

**VÌ SAO chỉ City mới chết:** Wild có vài chục building; **City có 6906 món nội thất** (`CityFurnisher` bày cho 536 nhà: 1960 rương nhỏ, 916 giường, 755 chậu cây…). Code M19 viết cho Wild, doc của nó còn ghi rõ *"quét trực tiếp mỗi lần gọi — pawn thật ít, chấp nhận được"* — giả định đó ĐÚNG ở Wild và SAI hoàn toàn khi B2 (2026-08-01) port pawn AI sang City qua `LocalMapHost`. **Bài học tái dùng: port một hệ sang map lớn hơn 100× thì mọi chỗ ghi "chấp nhận được vì ít" phải được rà lại.**

**Phương pháp đo (đáng lặp lại):** `Assets/Scripts/Tests/PerfProbe.cs` (MonoBehaviour tạm, gắn runtime qua MCP) + `ProfilerMarker` để lại vĩnh viễn trong `CityNpcSystem`/`CityNpcRenderer` + `PawnAiBrain.ProfileGivers` (static bool, mặc định TẮT) đo từng JobGiver.
- **⚠ BẪY ĐO: `execute_code` của MCP biên dịch C# NGAY TRÊN MAIN THREAD** (tốn 200ms–2.3s) ⇒ nếu probe còn lấy mẫu lúc gọi lệnh đọc thì CHÍNH PHÉP ĐO tạo ra đỉnh mà nó báo cáo (đã dính: max=1198ms trong khi p95=3.96ms). Bắt buộc `StartWindow(seconds)` tự tắt TRƯỚC khi đọc.
- **⚠ Unity không tick khi cửa sổ mất focus** → `Application.runInBackground = true`, nếu không treo ở `playmode_transition`.
- **⚠ Recompile giữa Play Mode làm domain reload** → bootstrap không chạy `Start()`, layout null. Phải Stop → compile → Play lại.
- **Nhìn p99/max/số frame quá ngưỡng, KHÔNG nhìn avg** — ở 1x avg 2.68ms trông hoàn hảo trong khi vẫn có 20 frame >100ms; giật là hiện tượng của ĐUÔI.
- **A/B tắt nguyên component** (`brain.enabled = false`) là cách khoanh vùng nhanh nhất, nhưng chỉ khoanh vùng — phải đo tiếp TỪNG giver mới chỉ đích danh.

**Test sau sửa:** `RegionPathfinder` 21/21, `Npc` 29/32 (3 fail cũ không đổi), `JobDriver` PASS, 0 lỗi console.

**CÒN LẠI:** user verify cảm nhận thật. `PerfProbe.cs` + `ProfileGivers` là đồ đo tạm — xoá được nếu vướng, nhưng nên giữ vì rẻ (marker gần như 0 khi profiler tắt) và chuỗi 5 vòng này chứng minh dự án cần đo thường xuyên. **Nghi vấn còn lại chưa đo:** `CityNpc.SlowTick` max 15.4ms (quét cả 874 dân mỗi giây in-game) — chưa thành vấn đề nhưng là chỗ tiếp theo nếu dân số tăng.

**Đo lại 2026-10-09** (CityScene Play Mode, 1000 dân, ×8, cửa sổ ~6h50–9h00 in-game, ~2.200 frame, ProfilerRecorder trên marker có sẵn): luồng chính cả game p50 4,5 / p95 7,4 ms. `CityNpc.Update` TB 0,36 ms/frame (p95 0,69) = StepOutdoors 0,20 + StepIndoorWalkers 0,11 + xử lý hàng đợi đường 0,05. `CityNpc.SlowTick` 3,9 ms MỖI LẦN (p95 5,4, max 11,5) — so 2,1 ms ngày 10-02 ⇒ tăng ~85% sau đợt hộ/bạn bè/nhu cầu; đây là chỗ dễ giật nhất. HourTick ~4 ms/giờ. A* Burst trên luồng phụ: 233 lần, TB 0,27 ms/lần (cache đường đi làm ăn phần lớn). Cách đo: `ProfilerRecorderHandle.GetAvailable` lấy handle theo tên marker (gồm "CellAStarJob (Burst)" cho job luồng phụ), cất recorder qua `AppDomain.SetData` giữa các lần execute_code; mỗi lần gọi execute_code chặn luồng chính ⇒ bỏ qua max của "Main Thread".

**Tách SlowTick 2026-10-09** (marker mới `CityNpc.Slow.*` + `NpcBrain.ChooseTask/FindPartner/FindFreeSpot`, để lại trong code): 280 lượt, TB 4,4 ms (p95 7,2). Nhu cầu thật (`TickActiveNeeds` → `PawnNeedsSystem.ApplyPassiveTick`) 1,7 ms = 39% — vì `_revealed` giờ có **763** người + 255 ngoài đường ⇒ gần cả thành chạy nhu cầu đầy đủ mỗi nhịp (thiết kế gốc là "tập nhỏ đang mở nóc"; mở hết nóc 2026-10-05 đã biến nó thành cả thành). Vòng não 2,3 ms = 54%: 1039 lần Refresh 1,3 ms, trong đó chỉ ~29 người/lượt thật sự rút việc mới nhưng mỗi lần ~22 µs (0,63 ms); phần còn lại ~1 ms là đường nhanh + tra 2 Dictionary chuỗi/người + chính marker. Gặp gỡ (scan) 0,26 ms, tìm bạn ~0. Có 1 đỉnh 46 ms (một ChooseTask 19 ms — nghi GC/JIT lần đầu, chưa xác minh).
**User chốt 2026-10-09:** chi phí hệ NPC hiện tại KHÔNG phải nợ kỹ thuật, không cần cắt (kể cả nhu cầu cho 763 người mở nóc). Đừng đề xuất tối ưu lại trừ khi dân số tăng mạnh hoặc có giật thật đo được.
