# Case study 08 — Viết lại bộ sinh địa hình cho thế giới 3D liền (19–21/10/2026)

**Kỹ năng:** sinh thủ tục (noise, spline, mật độ 3D), Burst, kiểm chứng bằng đo trong Play Mode, ra quyết định thiết kế cùng một trợ lý AI.

## Bối cảnh và quyết định
Bộ sinh cũ là của game 2D: núi = ô "Mountain" thành tường cao 3–7 khối; độ cao đắp sau; bản build thật vẫn chạy bộ sinh 2D một ô 200² vì công tắc thế giới liền chỉ bật trong Editor.

**Quyết định tôi chốt (không diễn giải lại):**
- Trục đứng **không giới hạn về kiến trúc**, nhưng khung *sinh* cố định Y 0..512; sửa 70 chỗ dùng cứng `BlockLayers.Count = 256`.
- Sông chảy thật từ mạng sông của world (rộng dần, hạ mặt nước theo dốc, cắt núi thành hẻm).
- Hình địa hình: world + **spline kiểu MC 1.18** + hồ sơ từng biome bằng **JSON kiểu Hytale**, trộn liền mép.
- Mực biển = Y 160. Biome: hàm tại *từng khối* từ cùng các trường sinh địa hình (nhiệt, ẩm, lục địa, xói mòn, đỉnh–thung), cỡ mảng kiểu MC; thú/sự kiện đọc biome *theo chunk*. Hướng dài hạn: **gỡ phụ thuộc bản đồ 2D 512²** — mọi thứ mới phải là hàm theo toạ độ + seed.

## Chuỗi commit (19/10 → 21/10)
Bỏ trần 256 lớp → hàm độ cao 3D + cửa sổ xem trước → **chạy hàm độ cao trong Burst, giống C# từng bit** → dời gốc về mực biển 160, lát nén 4 bit → "game feel" (núi thấp hơn, đồng bằng lượn, bậc thềm) → **sông và hồ thật** → cây giãn tán → biến đổi theo vùng (đồng phẳng để xây vs gồ ghề) → cửa sổ editor bản đồ → bỏ viền biển → uốn toạ độ + trục đỉnh–thung → **điêu khắc mật độ 3D** (nhô/trụ) → biome theo khối → khí hậu từ noise thuần → hang nhiều loại.

## Một bug được truy đúng nguyên nhân: "cột đá vô tri"
- Tôi báo (20/10): nhiều cột đá/cỏ vô nghĩa.
- Giả thuyết "hàm độ cao lỗi" → đo trong Play Mode: **không phải**.
- Nguyên nhân thật: điêu khắc 3D khoét 5–7 khối quanh một điểm, mặt đất gốc đứng lại thành cột phủ cỏ — **34/37** cột là cột như vậy, mọc thêm chỉ 3/37. Biên độ điêu khắc (~14×1,6 khối) ≫ độ dốc mặt đất.
- **Bẫy khi đo:** lần đầu đếm ra 525 "cột"/ô, thật ra là *cây*. Phải loại khối thân/lá khỏi phép đo. Mô phỏng ngoài game *không bắt được* — phải xem khối thật.
- Sửa: `SculptLimits` — chỉ nặn trong [đất thấp nhất, đất cao nhất] bán kính 2 ± 1. Kết quả: **0 cột nhô**.

## Hệ hang (21/10)
Hang lớn, buồng dẹp, khe rãnh nửa mở lên mặt, nước/dung nham ngầm. **Bất biến chính:** hai vùng chất lỏng khác mực ⇒ có vách đá ngăn ⇒ không có tường nước đứng, nước không chạm dung nham (kiểm bằng cặp cột kề, 25 ô: 0 tường chất lỏng). Khe gần nước/cây/công trình giữ mái → thành cầu tự nhiên, không cây lơ lửng. 17 test hang; chi phí +18% (đo nóng — *lần chạy lạnh trông như ×2,5 do JIT, đừng tin*).

## Ghi rõ những gì chưa giải quyết
- **Chi phí dựng ô +28%** (1049 → 1343 ms/ô, đo trên 25 ô) sau khi thêm điêu khắc/PV/tính đất. Chạy luồng nền nên không giật, nhưng ô mới hiện chậm ~0,3 s. Điêu khắc ≈ 150–200 ms/ô, còn ~100 ms chưa phân rã → đã ghi thành việc cho bên tối ưu, kèm ràng buộc: Burst ≡ C# từng bit, điêu khắc tất định theo toạ độ tuyệt đối.
- Biome tại khối: `Fill` ~93 ms/ô bằng C# thường → cần Burst hoặc lấy mẫu thưa hơn.

## Ảnh (cũ vs mới là phần quan trọng)
| Ảnh | Ghi chú |
|---|---|
| [`nui_cu_vs_moi`](../images/worldgen/nui_cu_vs_moi.png) | bản đồ độ cao núi: dải răng cưa cũ (trái) và đường đồng mức mượt hơn (phải) |
| [`dong_bang_cu_vs_moi`](../images/worldgen/dong_bang_cu_vs_moi.png), [`bo_bien_cu_vs_moi`](../images/worldgen/bo_bien_cu_vs_moi.png) | cùng kiểu so sánh |
| [`seam_59_60` → `_sau`](../images/worldgen/seam_59_60.png) | đường nối ô world (mountain seam) trước/sau khi chọn nền theo biome |
| [`trong_game_ho` → `_tuong_nuoc_da_sua`](../images/worldgen/trong_game_ho_tuong_nuoc_da_sua.png) | hồ trong game, sau khi sửa lỗi "tường nước" |
| [`trong_game_song_hem`](../images/worldgen/trong_game_song_hem.png) | sông chảy trong hẻm núi tuyết |

![Sông trong hẻm](../images/worldgen/trong_game_song_hem.png)
![Hồ](../images/worldgen/trong_game_ho_tuong_nuoc_da_sua.png)

## Bằng chứng
[worldgen_3d_redesign](../docs/lessons/project_worldgen_3d_redesign.md) · [sculpt_pillars_fix](../docs/lessons/project_sculpt_pillars_fix.md) · [cave_system](../docs/lessons/project_cave_system.md)
