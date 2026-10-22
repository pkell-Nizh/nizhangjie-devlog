# Case study 02 — Một dòng cấu hình: 2,8 → 92,6 FPS (và 27 → 280 FPS)

**Thời gian:** 01/08/2026 · **Kỹ năng:** đo A/B, profiling đồ hoạ 2D (Unity URP) · **Kết quả:** 367.899 → 70 draw call.

## Vấn đề
Bản đồ thành phố 400×400 chỉ chạy **2,8 FPS**.

## Cách tìm ra
- Đo A/B cùng scene, cùng phương pháp: `TilemapRenderer.Mode.Individual` cho **367.899 draw call** (gần đúng 1 ô = 1 draw call) so với `Mode.Chunk` 16×16 cho **70 draw call**.
- Kiểm tra `Individual` có lý do tồn tại không: **không** — comment cạnh nó chỉ giải thích phần shader, nhiều khả năng bị chép theo.
- Kiểm tra chức năng đi kèm vẫn đúng: tô màu từng ô (`SetColor`) vẫn chạy ở Chunk (màu nằm trong vertex color); xác minh bằng lọc 500 ô vai trò và overlay 160k ô.

| Scene | Individual | Chunk 16×16 |
|---|---|---|
| City 400×400 | 367.899 draw call · **2,8 FPS** | 70 draw call · **92,6 FPS** |
| Wild 59.815 ô | 58.434 draw call · 27,1 FPS | 159 draw call · **280,6 FPS** |

## Quyết định đáng chú ý
- **Chunk 16×16 thay vì 32×32 mặc định:** sửa 1 ô là rebuild cả mesh chunk chứa nó; Wild sửa ô liên tục (đánh dấu/xây/thu hoạch) nên chunk nhỏ = rebuild rẻ.
- **Một rủi ro Claude nêu ra hoá ra vô nghĩa:** Claude lo vấn đề sắp xếp vẽ (vật cao che pawn) khi dùng Chunk; tôi chỉ ra dự án *không có vật cao nào*, mọi thứ là ô màu phẳng nhìn từ trên xuống. → **Kiểm tra loại art đó có tồn tại chưa rồi hãy nêu nó như một rào cản.**
- Phát hiện phụ: scene Wild mặc định mở vào một ô nước sâu → map toàn nước khi mở trực tiếp (không lộ khi vào từ world map). Ghi lại, chưa sửa.
- Ý tưởng "nóc nhà che nội thất" được đánh giá lại: nó cắt ~⅓ số ô, ở chế độ Individual vẫn ~160k draw call nên không cứu FPS; giá trị thật là **dễ đọc bản đồ**.

## Bẫy đo
Dùng `UnityStats.frameTime` khi Editor mất focus đọc ra 0,15 ms (6600 FPS) — rác. Đúng: so `UnityStats.drawCalls` trước/sau, và FPS thật bằng cách đếm `Time.frameCount` giữa hai mốc đồng hồ.

## Bằng chứng
[`docs/lessons/project_tilemap_chunk_mode.md`](../docs/lessons/project_tilemap_chunk_mode.md) · ảnh 2D thời kỳ này: [`images/timeline`](../images/timeline) (03/08/2026)
