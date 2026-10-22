---
name: project-tilemap-chunk-mode
description: Mode.Individual là thủ phạm lag (1 ô = 1 draw call); City đã flip sang Chunk 16×16 — 2.8→92.6 FPS. Wild/WorldMap CHƯA flip
metadata: 
  node_type: memory
  type: project
  originSessionId: 4659abf3-759a-41e9-962d-cfd705ea995e
  modified: 2026-08-01T16:18:43.815Z
---

**Đo được 2026-08-01, A/B cùng scene cùng phương pháp (CityScene, map 400×400, zone overlay bật):**

| | draw calls | FPS |
|---|---|---|
| `TilemapRenderer.Mode.Individual` | 367,899 | **2.8** |
| `Mode.Chunk`, `chunkSize 16×16` | 70 | **92.6** |

Gần đúng **1 ô = 1 draw call** ở Individual. Đó là thuộc tính của thứ đang vẽ nên **không phụ thuộc Editor có focus hay không** — dùng `UnityEditor.UnityStats.drawCalls` để so trước/sau, ĐỪNG dùng `UnityStats.frameTime` (đọc ra 0.15ms = 6600 FPS khi Editor không focus, hoàn toàn rác). Muốn FPS thật thì đếm `Time.frameCount` giữa 2 mốc wall-clock.

**`Individual` chưa từng có lý do được ghi lại** — comment cạnh nó chỉ giải thích phần shader Unlit bên dưới, nhiều khả năng bị chép theo cùng lúc. `SetColor()` từng ô **vẫn chạy bình thường ở Chunk** (màu bake vào vertex color của mesh chunk) — đã verify: filter vai trò 500 ô ra đúng `RGBA(1, 0.95, 0.35, 0.55)`, zone overlay 160k ô vẫn đúng màu.

**ĐÃ FLIP:** `CityTilemapRenderer.ConfigureTilemapRenderer` (đổi tên từ `SetIndividualMode`) + `CityZoneOverlay.EnsureRendererMode`.

**ĐÃ FLIP LUÔN Wild + WorldMap** (`WildTilemapRenderer.ConfigureTilemapRenderer`, `WorldMapRenderer.ConfigureTilemapRenderer`). Đo ở WildLocal ô (120,120) Rừng Rậm, 59,815 ô (40k ground + 19.8k feature):

| | draw calls | FPS |
|---|---|---|
| Individual | 58,434 | 27.1 |
| Chunk 16×16 | 159 | 280.6 |

Feature (cây/quặng/crop) vẫn đúng màu dưới Chunk.

**Rủi ro sort tôi từng nêu là VÔ NGHĨA ở dự án này** — user chỉ ra: *không có vật cao nào cả*, mọi thứ đang là ô màu phẳng 1×1 nhìn từ trên xuống, không có gì che được pawn. Bài học: kiểm dự án có tồn tại loại art đó chưa rồi hãy nêu nó như rào cản. Và kể cả sau này có art cao thật, cách giải là **tách layer có `sortingOrder` riêng** hoặc để vật cao là `SpriteRenderer` sort theo Y — KHÔNG quay lại Individual.

**Vì sao 16×16 chứ không phải 32×32 mặc định:** sửa 1 ô là rebuild mesh CẢ chunk chứa nó. Wild sửa tile liên tục (designation/build/gather) nên chunk nhỏ = rebuild rẻ, đổi lại nhiều draw call hơn chút.

**Phát hiện phụ, CHƯA sửa:** `WildSceneBootstrap` mặc định `tileX=5, tileY=7` là **ô nước sâu** → mở thẳng scene WildLocal ra map toàn nước, `Features.Count = 0`. Vào từ world map thì `WorldEntryContext` ghi đè nên không thấy; chỉ lộ khi mở scene trực tiếp để test.

**Hệ quả cho ý tưởng "nóc nhà che nội thất":** nó cắt ~⅓ số ô, ở Individual vẫn còn ~160k draw call nên KHÔNG cứu được gì; giá trị thật của nó là **dễ đọc bản đồ** (mirror RimWorld), không phải FPS. Xem [[project_city_wild_ui_port]].
