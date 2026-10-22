# Những lần thiết kế sai — và cách giải quyết cùng nhau

> Đây là danh sách các **lỗi thiết kế / quyết định sai** của dự án, lấy từ ghi chú làm việc lúc đó. Với mỗi lỗi mình ghi: *sai ở đâu*, *ai/cái gì phát hiện ra*, *hướng sửa do ai đề xuất*, *kết quả*, *bài học*. "Người dùng" = mình (chủ dự án); "AI" = trợ lý lập trình (Claude Code). Chỗ nào ghi chú không nói rõ ai đề xuất thì mình ghi "không ghi rõ".
>
> Mục đích: cho thấy **cách phát hiện và sửa sai**, không phải một dự án hoàn hảo.

## Bảng tóm tắt
| # | Lỗi thiết kế | Phát hiện bởi | Bài học |
|---|---|---|---|
| 1 | Tối ưu nhầm hệ 4 lần liên tiếp | người dùng yêu cầu "đo profiler" | triệu chứng là dữ liệu, không phải chẩn đoán |
| 2 | Giả định "pawn không sống ở City" hết hạn | sự cố HUD.prefab | hỏi "khác biệt này có lý do kỹ thuật thật không?" |
| 3 | `Disciple` (tu tiên) khoá cứng vào lõi game | người dùng: "xianxia chỉ là một dịch vụ" | lõi nhận `PawnBody`, hỏi registry khi cần tu vi |
| 4 | Hai bảng thoại hardcode đại từ khác nhau | AI khi làm hệ thoại | cơ chế ở lõi, từ ngữ ở content-pack |
| 5 | Thêm một trục lọc = nhân đôi nợ nội dung | AI đo (192 ô → ~576 câu) | thêm trục phải trả lời "ai viết gấp đôi số câu" |
| 6 | Công trình người chơi xây biến mất khi quay lại ô | người dùng hỏi "lỗ hổng cơ" | trạng thái mới → hỏi "có sống qua save/load không" |
| 7 | Lưu game lưu bản *cũ* của ô đang đứng | AI khi nối save | chụp ô đang sống trước khi ghi |
| 8 | "Xong" khi mới có test xanh (nội thất vô hình) | người dùng: "đã thêm vào game để mình chơi thử chưa?" | "xong" = người chơi chạm được |
| 9 | Biome là nhãn dán 2D, quyết định *trước* địa hình 3D | người dùng | biome là hàm theo toạ độ + seed |
| 10 | Điêu khắc 3D tạo "cột đá vô tri" | người dùng báo; AI đo nguyên nhân | đo khối thật trong Play Mode |
| 11 | Hệ bậc công cụ (tier) | người dùng đề xuất "lực phá" | lực theo từng tag, công thức có sàn/trần |
| 12 | Luật "trong nhà không mô phỏng" | người dùng bỏ luật | giảm *nhịp* ở xa, không bỏ vật lý |
| 13 | Tường/cửa cũ là hệ riêng song song với khối | người dùng chốt gộp | không duy trì hai đường cho cùng một thứ |
| 14 | UI: layout làm text co về 0 (logic đúng, layout sai) | điều tra | tách "logic sai" khỏi "hiển thị sai" |
| 15 | Quyết định "chỉ xem" của panel sức khoẻ bị đảo | người dùng | chấp nhận đảo quyết định khi dùng thật |
| 16 | A* chạm trần + chặn công trình lệch với region | AI đo 400 cặp | một phần còn là câu hỏi thiết kế mở |
| 17 | Cache path nhốt NPC (null do bị cắt ngang) | AI tự bắt trước khi gây hại | chỉ cache kết quả *thành công* |
| 18 | Hoạt động Wild chạy ở City với giả định "ít" | đo Profiler | port hệ sang quy mô 100× → rà lại mọi "chấp nhận được" |

---

## 1. Bốn vòng tối ưu nhầm hệ (08/08)
- **Sai ở đâu:** đọc code rồi suy ra thủ phạm là pathfinding, trong khi triệu chứng "giật khi pathfinding sinh ra" chỉ là *trùng nhịp* với hệ khác.
- **Phát hiện:** người dùng bảo "đo profiler đi" → thủ phạm thật là `FindObjectsByType` trong vòng lặp lồng (77 ms/lần gọi, 97% chi phí AI).
- **Cùng giải:** người dùng đề xuất hai điều có giá trị: *thu hẹp đồ thị tìm đường tận gốc* (2369 → 590 region) và *chọn việc gần nhà*. Phần đo thật cho ra **713 ms → 3,26 ms**.
- **Bài học:** [case 01](case-studies/01-lag-thanh-pho-bon-lan-doan-sai.md); thành luật C3 "đo trước khi tối ưu".

## 2. Giả định "pawn không sống ở City" đã hết hạn (27/08)
- **Sai ở đâu:** HUD dùng chung hai scene nhưng thiết kế ngầm cho rằng pawn chỉ sống ở Wild.
- **Phát hiện:** sự cố Apply All/Revert làm Wild mất sạch phần riêng.
- **Cùng giải:** người dùng nhìn ra Wild và City *bản chất là một, chỉ khác bộ nạp dữ liệu* → sửa City cho bắt kịp thay vì dựng lại Wild cũ; AI đề ra phép thử "khác biệt này có lý do kỹ thuật thật không?" để quyết hợp nhất cái nào, giữ tách cái nào.
- **Bài học:** [case 03](case-studies/03-su-co-hud-prefab-hop-nhat.md).

## 3. `Disciple` khoá cứng vào lõi (M37, 15/08)
- **Sai ở đâu:** lớp nội dung tu tiên nằm ở 199 file; Companion và Beast phải *fork riêng đường ống AI* chỉ để né nó.
- **Phát hiện:** yêu cầu của người dùng — *"avatar, pawn, NPC không phụ thuộc disciple… xianxia chỉ là một nhà cung cấp dịch vụ"*; sau đó chốt lại 09/08 rằng mục tiêu là engine cho mọi nội dung.
- **Sửa:** `PawnBody` là lõi, `Disciple` thành lớp con (refactor không vỡ), tu vi tách vào registry; test "gỡ xianxia vẫn chạy".
- **Một lỗi của AI được ghi lại:** từng quên rằng xianxia chỉ là content pack → thành ghi chú "core sim trước xianxia".
- **Bài học:** [case 05](case-studies/05-tach-loi-khoi-tu-tien.md).

## 4. Hai bảng thoại hardcode đại từ khác nhau (M50, 21/08)
- **Sai ở đâu:** `memory_lines` hardcode "anh/tôi", `dialogue_lines` hardcode "ngươi/ta" → một pawn nói "ngươi" rồi "anh" ngay sau trong cùng một lượt.
- **Sửa:** giải đại từ *một lần* theo quan hệ rồi áp cho cả hai nguồn; test quét mọi câu để không còn "ngươi/mày/tao" hardcode.
- **Bài học:** [case 04](case-studies/04-he-loi-thoai-khong-lap.md).

## 5. Thêm trục lọc = nhân đôi nợ nội dung (M49→M50)
- **Sai ở đâu:** 6 trục lọc cho ra 192 ô; mỗi ô 3 câu = ~576 câu viết tay. Trục lọc thêm vào code chỉ mất 5 phút nhưng nợ nội dung là vĩnh viễn.
- **Sửa:** hai "đòn bẩy" không cần viết thêm chữ (chống lặp, đại từ theo quan hệ), rồi ghép mảnh; điểm dừng được đặt trước ("nếu 144 câu ghép đọc dở hơn 16 câu viết tay thì dừng").
- **M52** còn lộ hai bug thật: lines ngược nghĩa giữa hai bảng; ba nhịp dùng chung một dòng chống lặp.

## 6. Công trình biến mất khi quay lại ô (M61, 28/08)
- **Sai ở đâu:** mỗi lần vào lại một ô, bản đồ tái sinh hoàn toàn mới; địa hình tất định theo seed nên *trông* như cũ, chỉ nhà là mất — lỗi im lặng.
- **Phát hiện:** người dùng hỏi thẳng *"cái gì là lỗ hổng cơ?"* rồi yêu cầu vá.
- **Sửa:** lưu snapshot 6 loại, khôi phục *trước* khi dựng pathfinder (thử qua sự kiện `OnGenerated` trước thì sai thứ tự). Trước đó M41 cũng phát hiện **đệ tử chưa bao giờ được lưu**.
- **Bài học:** [case 13](case-studies/13-do-tin-cay-luu-tru-va-su-co.md); thành luật B9.

## 7. Lưu game ghi bản *cũ* (M61)
- **Sai ở đâu:** snapshot chỉ chụp lúc *rời* map → bấm Lưu giữa chừng lưu bản cũ/rỗng.
- **Phát hiện:** AI tự tìm ra khi nối `SaveManager`; vá bằng chụp ô đang sống ở đầu `WriteFile`. Ghi rõ: lỗi cùng loại ở *vị trí pawn* chưa vá.

## 8. "Xong" khi mới có test xanh (01/08)
- **Sai ở đâu:** AI báo nội thất thành phố "xong" vì test xanh, nhưng renderer chưa đọc dữ liệu → nội thất vô hình; hàm cung cấp chỉ được gọi trong test; không có đường chuột phải.
- **Phát hiện:** người dùng hỏi lại *"khoan, cậu đã thêm nó vào game để mình chơi thử chưa?"*. Đây là lần thứ ba dính bẫy "viết xong nhưng chưa gắn vào scene".
- **Sửa:** luật D1 "XONG = người chơi chạm được" và cách báo cáo "đã code + test xanh, CHƯA verify Play Mode". [case 14](case-studies/14-cach-lam-viec-voi-ai-va-bo-luat.md).

## 9. Biome là nhãn dán 2D (20/10)
- **Sai ở đâu:** biome gán lên ô world 2D 240 khối, quyết định *trước* địa hình 3D → không khớp thế giới khối.
- **Cùng giải:** người dùng quyết: biome là hàm theo toạ độ (nhiệt/ẩm/lục địa/xói mòn/đỉnh–thung); khí hậu từ noise thuần (không dải vĩ độ); thú/sự kiện đọc biome theo chunk; *"sau này phải gỡ 2D ra cho nhẹ"* → bước "nhãn world từ hàm mới" bị **bỏ**, KHÔNG thêm hệ mới đọc `Cell`.
- [case 08](case-studies/08-sinh-dia-hinh-3d.md).

## 10. "Cột đá vô tri" (20/10)
- **Phát hiện:** người dùng báo; giả thuyết "hàm độ cao lỗi" bị bác bỏ bằng đo.
- **Nguyên nhân thật:** điêu khắc 3D khoét quanh một điểm, để lại cột (34/37). Sửa bằng giới hạn theo đất lân cận → 0 cột nhô. Bẫy: đếm nhầm *cây* thành cột (525/ô).

## 11. Hệ bậc công cụ → "lực phá" (16/10)
- **Sai ở đâu:** công cụ = món mặc ở `core:weapon`, hotbar đoán loại bằng chuỗi `if` theo id, mỗi thao tác một đường riêng — không hợp khi game chuyển sang roleplay kiểu Minecraft.
- **Cùng giải:** người dùng đề xuất *bỏ bậc, dùng lực phá*; AI góp ý lực **theo từng tag** (để món lai không thành "vua") và công thức thời gian có sàn/trần; người dùng chốt: lực nhỏ hơn độ cứng vẫn phá được nhưng rất chậm và không rơi đồ. Hiện thực: `ItemUseDispatcher`, hotbar chọn = cầm.
- [kế hoạch gốc](docs/lessons/project_tool_item_plan.md).

## 12. "Trong nhà không mô phỏng" (18/10)
- **Sai ở đâu:** lối tắt cho sinh vật khác avatar (đi lưới, xuyên cửa, trượt thẳng trong nhà) — người chơi đọc ra thành lỗi.
- **Cùng giải:** người dùng: *"mọi sinh vật có vật lý y như avatar"* và *bỏ luật đó*. Nếu cần rẻ hơn thì giảm *nhịp* vật lý ở xa (60/20 Hz), không bỏ vật lý. Kẹt thì tìm đường lại, không teleport.
- [ghi chú](docs/lessons/project_creature_physics.md).

## 13. Hai đường song song cho tường/cửa
- **Sai ở đâu:** tường/cửa cũ (`BuildPiece`, "tầng tường") là hệ riêng bên cạnh khối.
- **Người dùng chốt:** gộp thành khối luôn, save cũ tự đổi — "bỏ đường riêng".
- [ghi chú](docs/lessons/project_shaped_blocks.md).

## 14. UI: logic đúng, layout sai (20/08)
- **Hiện tượng:** chi tiết panel sức khoẻ hiện ra như "ô vuông màu". Logic C# click-để-xổ-chi-tiết vốn **đúng**; nguyên nhân là `HorizontalLayoutGroup` với `childControlWidth=false` làm text co về 0 chiều rộng. Bài học: tách *logic sai* khỏi *hiển thị sai* trước khi sửa.

## 15. Quyết định "chỉ xem" bị đảo (M47)
- Tab Health ban đầu "chỉ xem"; sau khi dùng, người dùng đảo quyết định, thêm lại 3 nút (Priority/NoTend/Herb) ngay trên dòng. Ghi lại như một quyết định *được phép đổi* khi có dữ liệu sử dụng.

## 16. A* chạm trần + chặn công trình lệch với region (17/10)
- Xem [case 09](case-studies/09-tim-duong-canreach-findpath.md): giả thuyết "region nói dối" bị bác, sửa heuristic (87 → 11 lệch), và **phần còn lại** (ô có `Occupant` công trình chặn A* nhưng region bỏ qua) được ghi là *câu hỏi thiết kế còn mở*, không tự quyết.

## 17. Cache nhốt NPC (08/08)
- AI tự bắt **trước khi gây hại**: kết quả `null` do A* bị cắt trần bị cache hiểu là "vĩnh viễn không có đường" → NPC bị nhốt trong nhà. Sửa: chỉ cache đường *thành công*. Cùng đợt tự bắt: tường thành phố phá được lúc chơi nên cache phải vô hiệu theo `Regions.Version`, không theo toạ độ.

## 18. Giả định "ít" sai 100× khi port sang City
- Code M19 viết cho Wild ghi rõ "quét trực tiếp, pawn ít nên chấp nhận được"; City có **6906 món nội thất** → giả định sai ~100×. Luật: *port hệ sang quy mô lớn hơn 100× thì rà lại mọi chỗ ghi "chấp nhận được vì ít"*.

---

## Mẫu hình chung của cách sửa sai
1. **Người dùng thường là người nhìn ra sai ở tầng thiết kế** ("bản chất Wild và City là một", "xianxia chỉ là dịch vụ", "bỏ luật trong-nhà-không-mô-phỏng"); **AI thường là bên đo và tìm nguyên nhân kỹ thuật** (đếm 199 file, đo 400 cặp đường, đo 2700 lượt thoại).
2. **Luôn đo trước khi tin giả thuyết** — nhiều giả thuyết ban đầu của AI bị bác bỏ bằng số.
3. **Ghi giới hạn:** cái chưa vá, cái chưa tái hiện, cái cần người quyết.
4. **Mỗi lỗi thành một luật** trong [`CODE_SYSTEM_RULE`](docs/CODE_SYSTEM_RULE.md) hoặc một test canh ranh giới.

> *Về độ tin cậy:* mình viết lại từ ghi chú. Những chỗ ghi "người dùng chốt/đề xuất" là theo đúng lời ghi chú; mình không suy thêm. Nếu bạn nhớ khác, hãy sửa trước khi gửi nhà tuyển dụng.
