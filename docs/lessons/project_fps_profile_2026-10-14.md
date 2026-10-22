---
name: project_fps_profile_2026-10-14
description: Phân rã thời gian khung WildLocal + lớp 3D (2026-10-14) — thủ phạm script là ContextActionController quét mỗi khung; còn lại nghẽn ở chờ GPU; cách chụp Profiler qua execute_code
metadata:
  node_type: memory
  type: project
  originSessionId: d93f285a-5ead-48f8-9bcd-66fa9efac21c
  modified: 2026-10-15T08:19:06.011Z
---

Đo 2026-10-14, WildLocal 9 ô world (có thành phố: 6257 mảnh tường, 1690 công trình, 2515 feature), lớp 3D bật, 1x, cửa sổ Game 1100×500, qua MCP.

**Phân rã (Profiler bật, khung ~20 ms):** chờ GPU `WaitForLastPresentation` 5,2 ms · `ContextActionController.Update` 4,1 ms · EditorLoop 2,6 · `CompanionSpawner.Update` 1,3 · `VoxelGameView.LateUpdate` 0,8 · `VoxelWorld.Update` 0,7 · `CityNpcRender.Draw` 0,45. Tick KHÔNG phải thủ phạm (1x ⟂ 8x chỉ lệch ~1 ms/khung).

**Đã sửa (CHƯA commit):**
- `ContextActionController`: quét ô trước mặt chỉ khi ô nhắm đổi / mỗi 0,15 s / lúc bấm phím (trước: mỗi khung) ⇒ 4,1 → 0,5 ms/khung.
- `CompanionSpawner.Update`: dò "con chưa có token" 4 lần/giây, kể cả nhánh pending (cờ pending có thể đứng mãi khi chủ vắng mặt). CHƯA đo lại riêng — lần đo giữa chừng (chỉ sửa nhánh không-pending) vẫn 1,24 ms.
- `BlockPlacementOps.WallAt`: chỉ mục tường theo ô, dựng lại mỗi khung có hỏi (chỉ lúc Play; test Editor vẫn quét thẳng). Đối chiếu 2086 ô: 0 lệch.

**Kết quả (mỗi bản 1 lượt ~55 s):** 112 → 118,6 FPS; khung chính p50 8,0 → 6,2 ms; nhưng p95 13,3 → 18,4 ms (đuôi xấu hơn ở cả hai lượt sau sửa, CHƯA rõ nguyên nhân — có thể do nhịp quét dồn thành cú 1–4 ms, hoặc dao động giữa lượt).

**Còn lại / chưa làm:**
- `BlockPlacementOps.CanBreak`/`CanPlace` vẫn quét thẳng `BuildPieces` + `Features` + mọi `WildBuildings.Footprint` (0,3–3,6 ms mỗi lần ở thành phố) ⇒ cần chỉ mục theo ô cho feature và footprint công trình.
- Nghẽn lớn nhất còn lại là GPU/lưới: tổng 8,5 triệu đỉnh, chưa gộp mặt (greedy), chưa LOD.
- Avatar có thân vật lý chạy bước cố định 1/60 không nội suy ⇒ ở >60 FPS cứ ~2 khung có 1 khung đứng yên (khựng hình dù FPS cao) — chưa xác nhận với user, chưa sửa.

**Đo lại 2026-10-15 sau gộp mặt + ánh sáng trong shader (340d00b), cùng cảnh, Game 1100×500:** tổng 2068 lưới / 3,7 triệu đỉnh (trước 8,5 tr). Góc mặc định (nhìn chéo xuống): 150 chunk hiện, khung p50 5,1 ms (~195 FPS). Góc nhìn NGANG ra chân trời (orbit pitch 8°, xa 60): 826 chunk hiện / 1,36 tr đỉnh / 888 batch / 832 SetPass; khung p50 6,1 ms, p95 9,1 (có Profiler), ~159 FPS không Profiler. Chờ GPU chỉ còn 0,07 ms ⇒ HẾT nghẽn GPU (ở cỡ cửa sổ này); render CPU (ExecuteRenderQueueJob + SetPass) ~0,2 ms ⇒ lệnh vẽ chưa thành vấn đề. Script nặng nhất: `VoxelGameView.LateUpdate` 0,62 ms, `CityNpcRender.Draw` 0,31. ⇒ Loại chunk bị che / gom lệnh vẽ CHƯA cần; cần đo lại ở toàn màn hình / máy yếu trước khi làm.

**Bán kính ×2 (`LocalMapBootstrap.streamRadius` 1→2 = cửa sổ 5×5 ô world, `[Range(0,2)]` nên 2 là trần), 2026-10-15 — user chốt 2 là CHUẨN (code mặc định + WildLocal.unity = 2; máy test khá, không nhắm máy yếu):** 5619 lưới / 9,28 tr đỉnh, RAM cấp phát ~3 GB, bộ nhớ đồ hoạ 727 MB → 1137 MB khi nhìn ngang. Góc mặc định: p50 6,1 ms (gần như không đổi). Nhìn ngang: 1791 chunk hiện / 2,72 tr đỉnh / 1840 batch / 1707 SetPass; p50 8,4 ms, p95 11,7; ~124 FPS không Profiler (r1: 159). Chờ GPU 0,25 ms; `VoxelWorld.Update` 0,38→0,69 ms. Vẫn chưa nghẽn GPU ở 1100×500; thứ tăng theo bán kính là số lệnh vẽ và chi phí script. Cảnh báo "PlayerLoop called recursively" trong console là do chụp màn hình qua MCP lúc đang Play.

**Cách chụp Profiler qua execute_code:** `ProfilerDriver.ClearAllFrames(); profileEditor=false; Profiler.enabled=true; ProfilerDriver.enabled=true` → gọi lần sau: `ProfilerDriver.GetHierarchyFrameDataView(f, 0, MergeSamplesWithTheSameName, columnTotalTime, false)`, duyệt `GetItemChildren`, cộng `columnSelfTime` theo tên qua ~200 khung. Dữ liệu giữ qua `AppDomain.SetData` MẤT nếu Play bị khởi động lại giữa hai lần gọi.

Liên quan: [[feedback_measure_before_optimizing]], [[project_block_physics]], [[project_block_world]], [[project_kinematic_body]].
