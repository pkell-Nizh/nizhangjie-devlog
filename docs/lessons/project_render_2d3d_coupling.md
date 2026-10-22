---
name: project-render-2d3d-coupling
description: 2D ⟂ 3D phải là hai bản vẽ trên cùng dữ liệu — bước 1 (vị trí) + 2 (LocalMapData.Changed) + 3 (MapHighlight) commit f7f510e; bước 4 (2D vẽ khối) commit 3a5c869; bước 5 asmdef commit a3da2d0; đối chứng 2D #1 (3D chạy mọi map) + #2 (quyền lái camera) + #3 (bóng ném = `BallFlights`, nay thay bằng hệ vật bay [[project_projectile_system]]) + #4 (`IsCarrying`/`IsAngry`, nhãn 3D) commit c162fbe; #6 gục/chết (voxel `Lie` Side/Back từ IsDowned/IsDeceased) xong chưa commit; còn #5 mái, #7 xưởng vẽ, bước đóng kho 2D
metadata:
  node_type: memory
  type: project
  originSessionId: 61053c52-440a-438f-bf67-6cf98e68a510
  modified: 2026-10-11T19:15:42.117Z
---

User muốn tilemap 2D và lớp voxel 3D là HAI BẢN VẼ ngang hàng trên cùng một dữ liệu, không bản nào phụ thuộc trực tiếp bản kia.

**Đã làm 2026-10-12 (bước 1–3, test 18/18 + verify Play Mode, commit f7f510e):**
- Vị trí là dữ liệu: `WildPlayerToken.CellPos` (toạ độ ô số thực) + `DisplayColor`; `BeastView`/`CompanionView` có `pos`/`scale`/`color`. Sprite 2D chỉ chép theo. `BeastSpawner.Views` thay reflection. `WorldViewOverride.ScreenAnchor` nhận toạ độ ô.
- `LocalMapData.Changed` (`NotifyCell`/`NotifyStructure`/`NotifyFeatureCell`): sim và UI chỉ báo, không gọi `RenderCell`. `WildTilemapRenderer`/`CityTilemapRenderer`/`VoxelGameView` tự nghe. JobGiver và `PawnAiBrain` không còn cầm renderer 2D. Bỏ `BlockStore.Touch`.
- Bước 3: `World/UI/MapHighlight` (ô sáng xem trước là dữ liệu); `WildSceneInteraction` ghi vào, renderer 2D Wild/City + `VoxelGameView` cùng vẽ theo. `LocalMapBootstrap.StreamWorld` thay reflection.
- Test canh: `Run Render Separation Tests` (quét mã nguồn: ngoài renderer cấm gọi vẽ lại; `VoxelGameView`/`VoxelPovHud` cấm đọc sprite/transform).

- Bước 4 (3a5c869): `WildTilemapRenderer` nghe `BlockStore` và vẽ khối cao nhất của cột đè lên lớp `_buildPieces` (không có thứ tự trống giữa tường 3 và ô sáng 4). Test 24/24 + verify Play Mode.

- Bước 5 (a3da2d0): asmdef `Nizhangjie` (Scripts) + `Nizhangjie.Editor` (+ .asmref ở mọi thư mục Editor/) + `Nizhangjie.VoxelView` (+ .Editor); game không tham chiếu được voxel. 75 `m_TargetAssemblyTypeName` trong HUD.prefab + 4 scene đổi sang `Nizhangjie` (YAML có thể xuống dòng giữa kiểu và tên assembly!). `CombatDriverRegistry` lọc thêm đuôi `.Editor`. ⚠ `Run ALL Tests` làm Unity crash — chạy từng bộ.

**Còn lại:** user đang cân nhắc BỎ HẲN 2D, chỉ giữ 3D toàn tri (lộ trình đề xuất: voxel thành lớp vẽ chính chạy cả map thường/bí cảnh/Settlement → bù đồ rơi/tint phòng/xưởng vẽ → tắt 2D chơi thử → xoá 41 file). Token vẫn là một MonoBehaviour gánh cả thực thể lẫn sprite 2D; hai lớp vẽ dùng chung camera và HUD; HUD cũ của City vẫn tô highlight thẳng (chỉ City). Mẹo verify: menu chạy qua MCP ngay sau refresh có thể chạy trên domain cũ — gọi hàm test bằng `execute_code` + `logMessageReceived` đáng tin hơn `read_console`.

**Đối chứng chức năng đặc thù 2D (rà 2026-10-12, chưa sửa) — để đưa voxel 3D thành lớp vẽ chính, 2D "đóng kho" làm bản đối chiếu:**
- Đã có móc 3D: chọn ô/kéo-thả/ô đang trỏ/kính mái dùng `PointerCell.FromScreen` (Override); bong bóng + số sát thương (`WorldViewOverride`); nhãn cạnh avatar (`ScreenAnchor`); ngày đêm + thời tiết (3D `ApplySky` nhân màu vào khối, overlay 2D tự ẩn khi camera phối cảnh); đồ rơi/vùng/kho/đánh dấu/bản vẽ (mark riêng của 3D).
- Cần sửa: (1) ✅ XONG — 3D chạy map thường/bí cảnh (một hình W×H, độ cao từ `SourceTile` bọc WorldMap 1×1); (2) ✅ XONG phần giành camera — `WorldViewOverride.OwnsCamera`, `PlayerControlModeController` đứng ngoài khi 3D giữ camera; còn lại của camera: `LocalMapCamera` ortho, `PlayerControlModeController` Cinemachine 2D + Confiner2D, `WildPartyTestSpawner.SnapTo`; (3) ✅ XONG — `BallFlights` dữ liệu, voxel vẽ cầu + thanh tụ lực; (4) ✅ XONG — `IsCarrying`/`IsAngry`, nhãn voxel; (5) kính mái `WildRoofLens` + tint phòng chỉ 2D, 3D toàn tri bỏ hẳn mái; (6) ✅ XONG — voxel nằm nghiêng (gục, phập phồng) / nằm ngửa (chết), nhãn "(gục)"; (7) xưởng vẽ TemplateWorkshop chỉ 2D.
- Đóng kho cùng 2D, không port: City legacy (CityScene, CityHUDController, CityRoofLens, CityMinimapRenderer, CityCursorReadout, JobIconUI) — scene City đã chết từ M63. Lữ khách/dân thành phố có bản 3D riêng (VoxelTravelerRenderer, VoxelCityNpc* do phiên kia làm).

**Why:** đây là mục tiêu kiến trúc user đặt ra; không được làm sâu thêm sự dính nhau.
**How to apply:** code mới đọc vị trí từ `CellPos`/`pos`, không đọc `transform`/`SpriteRenderer`; đổi nội dung map thì gọi `data.Notify*`, không gọi renderer. Ô sáng/xem trước thì ghi `MapHighlight`, không gọi `renderer.Highlight*`. Thêm lớp vẽ mới thì nghe `LocalMapData.Changed` + `GridSystem.CellChanged` + `BlockStore.Changed` + `MapHighlight.Changed`. Liên quan [[project-block-placement]], [[project-control-scheme-split]].

**2026-10-13 — user chốt: bản 2D đã ĐÌNH TRỆ, không cập nhật nữa.** Tính năng mới (thế giới khối, đào/đặt mọi khối…) KHÔNG cần làm cho 2D, không cần giữ 2D đồng bộ. Chỉ cần đừng cố ý làm vỡ biên dịch.
