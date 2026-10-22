# Nhật ký những lần thiết kế sai — và cách tôi cùng Claude sửa

> Tôi viết lại từ ghi chú làm việc của chính dự án. "Tôi" là chủ dự án, "Claude" là trợ lý AI lập trình (Claude Code) làm việc cùng tôi. Tôi quyết định hướng đi và đặt tiêu chuẩn "xong"; Claude đo, viết code, chạy test và ghi tài liệu. Chỗ nào ghi chú không nói rõ ai nghĩ ra ý tưởng, tôi không nhận thêm công ở đó.
>
> Mục đích: cho thấy cách chúng tôi phát hiện và sửa sai, không phải một dự án chưa từng sai.

## Mục lục nhanh
| # | Lỗi thiết kế | Cách phát hiện | Điều tôi rút ra |
|---|---|---|---|
| 1 | Tối ưu nhầm hệ 4 lần | tôi dừng đoán, quyết định đo Profiler | triệu chứng là dữ liệu, không phải chẩn đoán |
| 2 | Giả định "pawn không sống ở City" hết hạn | sự cố HUD.prefab | hỏi "khác biệt này có lý do kỹ thuật thật không?" |
| 3 | Lớp tu tiên khoá cứng vào lõi game | tôi nhận ra dự án phải là engine | lõi nhận `PawnBody`, hỏi registry khi cần tu vi |
| 4 | Hai bảng thoại hardcode đại từ khác nhau | Claude lúc làm hệ thoại | cơ chế ở lõi, từ ngữ ở content pack |
| 5 | Thêm một trục lọc = nhân đôi nợ nội dung | Claude đo: 192 ô → ~576 câu | thêm trục phải biết ai viết gấp đôi số câu |
| 6 | Công trình biến mất khi quay lại ô | tôi hỏi "lỗ hổng cơ" | trạng thái mới → hỏi "có sống qua save/load không?" |
| 7 | Lưu game ghi bản *cũ* của ô đang đứng | Claude khi nối save | chụp ô đang sống trước khi ghi |
| 8 | Báo "xong" khi mới có test xanh | tôi hỏi "đã thêm vào game để mình chơi thử chưa?" | "xong" = người chơi chạm được |
| 9 | Biome là nhãn dán 2D | tôi thấy nó lỗi thời với thế giới khối | biome là hàm theo toạ độ + seed |
| 10 | "Cột đá vô tri" từ điêu khắc 3D | tôi thấy khi chơi; Claude đo nguyên nhân | đo khối thật trong Play Mode |
| 11 | Hệ bậc công cụ (tier) | tôi đề xuất đổi sang "lực phá" | lực theo từng tag, có sàn/trần |
| 12 | Luật "trong nhà không mô phỏng" | tôi quyết định bỏ | giảm nhịp ở xa, không bỏ vật lý |
| 13 | Tường/cửa là hệ song song với khối | tôi chốt gộp | không giữ hai đường cho cùng một thứ |
| 14 | UI: logic đúng, layout sai | Claude điều tra | tách "logic sai" khỏi "hiển thị sai" |
| 15 | Panel sức khoẻ "chỉ xem" | tôi đảo quyết định khi dùng thật | sẵn sàng đổi khi có dữ liệu dùng |
| 16 | A* chạm trần, lệch với region | Claude đo 400 cặp đường | một phần là câu hỏi thiết kế mở |
| 17 | Cache path nhốt NPC | Claude tự bắt trước khi gây hại | chỉ cache kết quả thành công |
| 18 | Giả định "ít" sai 100× khi sang City | đo Profiler | port sang quy mô lớn → rà lại mọi "chấp nhận được" |

---

## 1. Bốn vòng tối ưu nhầm hệ (08/08)
Thành phố giật vào giờ đi làm. Tôi và Claude đọc code, cùng kết luận thủ phạm là pathfinding, rồi sửa bốn vòng liên tiếp: né nội thất, cache đường đi, thu hẹp đồ thị, đa luồng. Vòng nào cũng giảm lag một chút, nhưng vẫn còn lag.

Đến đó tôi quyết định dừng đoán và đo Profiler. Kết quả lật ngược giả thuyết: 97% chi phí AI nằm ở một hàm không liên quan, quét toàn scene trong vòng lặp lồng (77 ms mỗi lần gọi). Sửa đúng chỗ, khung hình chính từ **713 ms xuống 3,26 ms**.

Trong chuỗi này, ý "thu hẹp đồ thị tìm đường tận gốc" (2369 → 590 region) là của tôi, và nó vẫn đáng giữ dù không phải thủ phạm chính. **Điều tôi rút ra:** triệu chứng người chơi mô tả là dữ liệu, không phải chẩn đoán. → [case 01](case-studies/01-lag-thanh-pho-bon-lan-doan-sai.md)

## 2. Giả định "pawn không sống ở City" đã hết hạn (27/08)
HUD dùng chung hai scene, nhưng thiết kế ngầm cho rằng pawn chỉ sống ở Wild. Một lần bấm Apply All rồi Revert làm Wild mất sạch phần riêng.

Tôi quyết định không dựng lại trạng thái cũ. Wild và City bản chất là một, chỉ khác bộ nạp dữ liệu, giống chunk trong Minecraft; thứ cần đổi là City. Claude đề ra phép thử để chọn cái gì hợp nhất, cái gì giữ tách: *"khác biệt này có lý do kỹ thuật thật, hay chỉ là tình cờ chưa ai dọn?"* → [case 03](case-studies/03-su-co-hud-prefab-hop-nhat.md)

## 3. Lớp tu tiên khoá cứng vào lõi (15/08)
Tôi nhận ra mục tiêu của mình không phải "một game tu tiên" mà là một engine mô phỏng cho mọi loại nội dung. Tôi quyết định tu tiên chỉ là một dịch vụ gỡ được.

Claude khảo sát trước khi động vào: lớp `Disciple` xuất hiện ở **199 file**, và hai hệ khác đã phải nhân bản đường ống AI chỉ để né nó. Chúng tôi chọn cách không phá code cũ: `Disciple` thành lớp con của `PawnBody`, tu vi tách vào sổ riêng, và một test "gỡ tu tiên vẫn chạy" canh biên giới. Cũng trong chuỗi này, Claude từng quên rằng tu tiên chỉ là content pack, nên tôi ghi thành nguyên tắc "core sim trước xianxia". → [case 05](case-studies/05-tach-loi-khoi-tu-tien.md)

## 4. Hai bảng thoại hardcode đại từ khác nhau (21/08)
Một pawn nói "ngươi" ở câu chào rồi "anh" ở câu ký ức ngay sau, vì hai bảng thoại hardcode hai kiểu xưng hô. Chúng tôi chuyển sang giải đại từ **một lần** theo quan hệ rồi áp cho cả hai nguồn, và thêm test quét mọi câu để không còn "ngươi/mày/tao" cứng. → [case 04](case-studies/04-he-loi-thoai-khong-lap.md)

## 5. Thêm một trục lọc = nhân đôi nợ nội dung
6 trục lọc cho ra 192 ô; mỗi ô 3 câu là ~576 câu viết tay. Thêm trục trong code chỉ mất 5 phút nhưng nợ nội dung là vĩnh viễn. Claude đo ra con số này, và tôi chọn hai "đòn bẩy" không cần viết thêm chữ nào (chống lặp, đại từ theo quan hệ), rồi mới ghép mảnh. Chúng tôi đặt trước một điểm dừng: nếu câu ghép đọc dở hơn câu viết tay thì dừng. Ở bước nhiều nhịp (M52) còn lộ thêm hai bug thật: hai bảng có lines ngược nghĩa, và ba nhịp dùng chung một dòng chống lặp.

## 6. Công trình biến mất khi quay lại ô (28/08)
Mỗi lần vào lại một ô, bản đồ tái sinh mới. Địa hình tất định theo seed nên *trông* như cũ, chỉ nhà người chơi dựng là mất, nên không ai thấy ngay. Tôi hỏi thẳng Claude "cái gì là lỗ hổng cơ" rồi quyết định vá. Claude dựng snapshot 6 loại, và phát hiện thứ tự phải là khôi phục *trước* khi dựng pathfinder. Trước đó, một lần rà khác (M41) cũng cho thấy **đệ tử chưa bao giờ được lưu**. → [case 13](case-studies/13-do-tin-cay-luu-tru-va-su-co.md)

## 7. Lưu game ghi bản *cũ* (28/08)
Snapshot chỉ chụp lúc rời map, nên bấm Lưu giữa chừng sẽ ghi bản cũ hoặc rỗng. Claude tự tìm ra khi nối save và vá bằng cách chụp ô đang sống ở đầu hàm ghi. Chúng tôi cũng ghi rõ: lỗi cùng loại ở vị trí pawn **chưa vá**.

## 8. Báo "xong" khi mới có test xanh (01/08)
Claude báo nội thất thành phố "xong" vì test xanh. Tôi hỏi lại: *"khoan, cậu đã thêm nó vào game để mình chơi thử chưa?"* Hoá ra nội thất sinh ra nhưng vô hình (renderer chưa đọc dữ liệu), hàm cung cấp chỉ được gọi trong test, và không có đường chuột phải. Đây là lần thứ ba dự án dính bẫy "viết xong nhưng chưa gắn vào scene". Từ đó tôi đặt luật: "xong" nghĩa là người chơi chạm được, nếu chưa thì báo thẳng "đã code + test xanh, CHƯA verify Play Mode". → [case 14](case-studies/14-cach-lam-viec-voi-ai-va-bo-luat.md)

## 9. Biome là nhãn dán 2D (20/10)
Biome gán lên ô world 2D quyết định *trước* địa hình 3D, nên không khớp thế giới khối. Tôi quyết định biome phải là hàm theo toạ độ và seed (nhiệt, ẩm, lục địa, xói mòn, đỉnh–thung), khí hậu từ noise thuần, thú và sự kiện đọc biome theo chunk. Tôi cũng quyết định về lâu dài phải gỡ phụ thuộc vào bản đồ 2D, nên bước "dán nhãn world từ hàm mới" bị bỏ. → [case 08](case-studies/08-sinh-dia-hinh-3d.md)

## 10. "Cột đá vô tri" (20/10)
Tôi thấy nhiều cột đá vô nghĩa khi chơi. Claude kiểm tra giả thuyết "hàm độ cao lỗi" và bác bỏ nó bằng số đo. Nguyên nhân thật là điêu khắc 3D khoét quanh một điểm và để lại cột (34/37 trường hợp). Sửa bằng giới hạn theo đất lân cận, còn 0 cột nhô. Một bẫy: lần đầu đếm ra 525 "cột" mỗi ô thật ra là *cây*.

## 11. Hệ bậc công cụ → "lực phá" (16/10)
Công cụ vốn là món mặc ở ô vũ khí, hotbar đoán loại bằng chuỗi `if` theo id, mỗi thao tác một đường riêng; không hợp khi game chuyển sang cầm-là-dùng kiểu Minecraft. Tôi đề xuất bỏ bậc và dùng lực phá. Claude góp ý tính lực **theo từng tag** để món lai không thành "vua". Tôi chốt thêm: lực nhỏ hơn độ cứng vẫn phá được nhưng rất chậm và không rơi đồ. → [kế hoạch gốc](docs/lessons/project_tool_item_plan.md)

## 12. "Trong nhà không mô phỏng" (18/10)
Đây là lối tắt cho sinh vật khác avatar (đi lưới, xuyên cửa), và người chơi sẽ đọc nó thành lỗi. Tôi quyết định bỏ luật này: mọi sinh vật có vật lý như avatar. Nếu cần rẻ hơn thì giảm nhịp ở xa (60 → 20 Hz), không bỏ vật lý; kẹt thì tìm đường lại, không teleport. → [ghi chú](docs/lessons/project_creature_physics.md)

## 13. Hai đường song song cho tường/cửa
Tường và cửa cũ (`BuildPiece`) là hệ riêng bên cạnh khối. Tôi chốt gộp thành khối luôn, save cũ tự đổi. → [ghi chú](docs/lessons/project_shaped_blocks.md)

## 14. UI: logic đúng, layout sai (20/08)
Chi tiết panel sức khoẻ hiện ra như "ô vuông màu". Logic click-để-xổ-chi-tiết vốn đúng; nguyên nhân là `HorizontalLayoutGroup` với `childControlWidth=false` làm text co về chiều rộng 0. Bài học: tách *logic sai* khỏi *hiển thị sai* trước khi sửa.

## 15. Quyết định "chỉ xem" bị đảo
Tab Health ban đầu chỉ để xem. Sau khi dùng thật, tôi đổi ý và thêm lại ba nút (Priority/NoTend/Herb) ngay trên dòng. Tôi coi đó là quyết định *được phép đổi* khi có dữ liệu sử dụng.

## 16. A* chạm trần, lệch với region (17/10)
Claude đo 400 cặp điểm: 87/252 cặp "tới được" nhưng không có đường. Giả thuyết "region nói dối" bị bác bỏ; sửa heuristic đưa số lệch xuống 11. Phần còn lại (ô có công trình chặn A* nhưng region bỏ qua) chúng tôi để là **câu hỏi thiết kế mở**, chưa tự quyết. → [case 09](case-studies/09-tim-duong-canreach-findpath.md)

## 17. Cache nhốt NPC (08/08)
Claude tự bắt lỗi này trước khi gây hại: kết quả `null` do A* bị cắt trần bị cache hiểu là "vĩnh viễn không có đường", sẽ nhốt NPC trong nhà. Sửa: chỉ cache đường thành công. Cùng đợt, nhận ra tường thành phố phá được lúc chơi nên cache phải vô hiệu theo phiên bản region, không theo toạ độ.

## 18. Giả định "ít" sai 100× khi sang City
Code của một hệ AI ghi sẵn "quét trực tiếp, pawn ít nên chấp nhận được". Khi chuyển sang City, có **6906 món nội thất**, giả định sai ~100 lần. Từ đó tôi đặt thói quen: đưa một hệ sang quy mô lớn hơn thì rà lại mọi chỗ ghi "chấp nhận được vì ít".

---

## Điều tôi thấy lặp lại
1. Tôi thường là người nhìn ra sai ở tầng thiết kế ("Wild và City là một", "tu tiên chỉ là dịch vụ", "bỏ luật trong-nhà-không-mô-phỏng"); Claude thường là bên đo và tìm nguyên nhân kỹ thuật (đếm 199 file, đo 400 cặp đường, đo 2700 lượt thoại).
2. Giả thuyết ban đầu, của cả hai chúng tôi, nhiều lần bị bác bỏ bằng số đo.
3. Chúng tôi luôn ghi lại giới hạn: cái chưa vá, cái chưa tái hiện, cái cần tôi quyết.
4. Mỗi lỗi thành một luật trong [`CODE_SYSTEM_RULE`](docs/CODE_SYSTEM_RULE.md) hoặc một test canh ranh giới.
