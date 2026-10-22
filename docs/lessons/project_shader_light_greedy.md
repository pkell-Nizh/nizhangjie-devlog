---
name: project-shader-light-greedy
description: 2026-10-15 ánh sáng + bóng góc chuyển sang Texture3D mỗi chunk để gộp mặt (greedy) — đỉnh 7,58→3,36 triệu; commit 340d00b; lý do + bẫy
metadata:
  type: project
---

User chọn (2026-10-15) "chuyển ánh sáng sang shader" sau khi đo: greedy giữ ánh sáng theo đỉnh chỉ gộp được ~16% (ánh sáng mịn + bóng góc nướng vào đỉnh làm mặt kề hiếm khi giống), chỉ-theo-texture thì −56%.

**Cơ chế** (`Prototypes/VoxelView/VoxelMapMesher.cs`, `VoxelView.shader`)
- `ShaderLight` const = true. `BuildLightVolume`: mỗi Acc lưới chunk (cả lưới hang) có Color32[] 18×H×18 (chunk + 1 viền, tầng theo min/max đỉnh ±1). R,G = trời/khối·w, B = w (ô có nghĩa sáng: không phải khối đầy; khối có hình chỉ tính khi mang sáng), A = độ thoáng (không phải khối đầy). Shader chia R,G cho B ⇒ trung bình chỉ trên ô trống, GIỐNG `LitVertex` (không lọt sáng ngoài vào nhà); AO = lerp(0.5,1,(A−0.25)/0.75).
- `ToChunkMesh` tạo Texture3D (Bilinear, Clamp, nhả CPU) lưu trong `_vols[mesh]`; `DestroyChunkMesh` huỷ cả hai; `ApplyLight(renderer, mesh)` đặt MaterialPropertyBlock (`_LightVol`, `_LightVolOrigin`, `_LightVolSize`, `_UseLightVol`). `VoxelGameView.SpawnChunk` gọi nó cho lưới chunk + hang; lưới nước/hộp rời vẫn dùng ánh sáng theo đỉnh (`_UseLightVol`=0).
- Lệch màu từng ô (Jitter) chuyển vào shader (băm ô = floor(op − n·0,5)), mesher bỏ Jitter khi ShaderLight.
- Greedy: `BeginGreedy` / `Defer(dir, lx, layer, lz, tile, color)` / `FlushGreedy` — lưới tạm ThreadStatic 6×16×16×256 ulong, khoá = tile<<24|rgb; chỉ mặt nguyên ô của khối đầy (dải cỏ trên hông vẫn phát lẻ).
- Test `Nizhangjie/Prototype/Run Voxel Greedy Tests` (11, reflection vào hàm riêng).

**Bẫy đã dính**
- Pháp tuyến từ ddx/ddy của toạ độ THẾ GIỚI (~7·10⁴) nhiễu ⇒ chấm đen (lấy mẫu vào khối đặc) + răng cưa texture nhìn gần. Nay tính pháp tuyến + UV theo toạ độ LƯỚI (`op`, object space, ô world đặt ở bội số nguyên nên UV y hệt) và chốt về trục gần nhất.
- Khi verify bằng dịch camera tay: mặt trong nhà / hang là lưới hang, chỉ dựng quanh camera ⇒ phải gọi `UpdateCaveMeshes` sau khi dịch, không thì thấy xuyên tường (không phải lỗi).
- Số đo: Texture3D 1583 cái ≈ 43 MB (RGBA32); dựng khối sáng ≈ 6% thời gian phát mặt.

**Còn có thể làm**: ánh sáng đổi (đuốc) hiện vẫn dựng lại cả lưới chunk (`UpdateLightRebuilds` → `OnChunkChanged`) — nay chỉ cần dựng lại Texture3D (rẻ hơn nhiều) trừ lưới nước; RGBA4444 nếu cần giảm bộ nhớ một nửa. Chưa đo FPS.
