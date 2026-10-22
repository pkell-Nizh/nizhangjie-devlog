# Chiến lược ra mắt: chọn gói nội dung nào để mở màn, và làm sao quảng bá mà không "sơn vòng ngắm lên trán"

**Thời gian:** 07/09/2026 · **Loại:** thảo luận chiến lược cùng Claude, **chưa phải kế hoạch đã thực hiện** · **Kỹ năng:** tư duy sản phẩm, phân tích rủi ro (pháp lý, nội dung, cộng đồng).

> Đây là phần "kinh tế" của dự án: không phải giá bán hay doanh thu, mà là *cách ra mắt thế nào để có người chơi, có cộng đồng, và không bị chặn*. Bài này tóm tắt một cuộc bàn kéo dài, ghi rõ ý của tôi và các phản biện của Claude.

## 1. Câu hỏi
Dự án có ba gói nội dung đã làm một phần: **Pokémon** (sưu tập sinh vật), **Tensura** (đặt tên, dựng quốc gia quái vật) và **Tu tiên**. Bỏ qua công sức đã bỏ vào tu tiên, **gói nào nên đi đầu khi ra mắt Early Access?**

## 2. Tôi chọn Pokémon trước — vì sao
- **Cộng đồng "đói ăn":** fan game Pokémon luôn phải tự làm từ đầu đến cuối (tự dựng thành phố, bản đồ, cốt truyện). Chỉ cần hé lộ là cộng đồng sôi sục, nhất là trên YouTube và Twitter.
- **Tensura:** không nên dùng tên gốc, vì cân bằng game sẽ làm lệch nguyên tác, điều cấm kỵ với người hâm mộ. Chủ đề chưa ai làm cũng mất nhiều thời gian để lan truyền.
- **Tu tiên:** tôi thừa nhận nó nhỏ hơn đối thủ lớn (Amazing Cultivation Simulator), nhưng thấy cơ hội ở chỗ thành phố, đơn giản hơn, và nhắm cả cộng đồng phiêu lưu lẫn quản lý. Hệ thống mod dựng lại truyện nổi tiếng có thể là nguồn nội dung quảng bá vô tận.
- **Lý do kỹ thuật:** cần dựng *sự sống động của thế giới* trước, chưa cần quá tập trung vào combat.

Claude đồng ý và nêu lý do mạnh hơn: thể loại sưu tập sinh vật là cái duy nhất trong ba cái cho phép ship một sim còn dang dở mà vẫn là một game hoàn chỉnh. Vòng lặp bắt → nuôi → đánh → khám phá tự đứng được. Nó cho **một cái sàn để bán** trong khi mình xây **cái trần**. Claude cũng đề xuất xếp **tu tiên vào gói 1.0+, không phải bản Early Access** (vì cần tầng mod đã chín và cộng đồng có sẵn), và **giữ cơ chế Đặt Tên của Tensura** như bản sắc riêng nhét vào gói sinh vật, không phí công đã làm.

## 3. Ba rủi ro Claude nêu ra (và cách tôi xử lý)
| Rủi ro | Cách xử lý |
|---|---|
| **Bản quyền.** Fan game Pokémon thường bị tắt đúng lúc nổi. | Quả bóng bắt phải là vật thể riêng có tên và hình riêng; không dùng chữ "Pokémon" ở kênh chính thức; để cộng đồng tự so sánh. Trong code: từ vựng "IP-safe" bằng tiếng Anh, công cụ thu phục không móc dữ liệu bóng nào có sẵn, và bộ 149 loài Gen 1 chỉ để **dev, bị loại khỏi bản public** (gitignore). |
| **Roster cạn nhanh.** Cộng đồng sẽ ngốn hết thú trong một cuối tuần. | Ship luôn *định dạng thú + editor + pipeline mod* như một tính năng của bản EA, kèm khoảng 25–30 con của chính mình. Tầng mod phải có thật lúc ra mắt, không phải lời hứa. |
| **Sim vô hình trong ảnh chụp.** Hệ thống không lên hình. | Cần vài "mồi" nhìn thấy được do chính hệ thống đẻ ra: tình cảm NPC, một con thú làm nghề buồn cười, một sự kiện thị trấn. |

## 4. Ý tưởng "greybox" của tôi
Để quảng bá an toàn, tôi đề xuất hé lộ bằng một **nhân vật đen không mặt**, ném một **khối lập phương đen** vào mấy con dummy làm chúng bị nhốt lại. **Không nhạc Pokémon, không tiếng Poké Ball.** Nội dung tiếp theo: màn hình chiến đấu cách tân, thuê nhà sinh sống, nuôi dummy, cửa hàng, khám bệnh, quan hệ NPC, hẹn hò. Fan tự ghép artwork vào, giống Cobblemon.

Claude đánh giá chiến lược này thông minh: nó phát tín hiệu "đây là framework" và không có nội dung Pokémon nào để bị đòi gỡ. Nhưng nêu 4 chỗ cần làm rõ:
1. **"Không mặt" là một giai đoạn hay bản sắc vĩnh viễn?** Cần sẵn một giai đoạn 2 "đây là game thật", vì Cobblemon miễn phí còn game của tôi bán trên Steam.
2. **Greybox chỉ bảo vệ phần quảng bá, không bảo vệ game khi ship.** Rủi ro pháp lý nằm ở *cơ chế* (ném để bắt), nên cơ chế bắt cũng cần khác hẳn, không chỉ hình dạng.
3. **"Màn hình chiến đấu cách tân" chưa có sẵn.** Cần quyết chiến đấu diễn ra trong thế giới hay vẫn là màn riêng.
4. Chiến lược này **hứa một bề mặt mod cỡ Cobblemon** ngay lúc ra mắt.

## 5. Ý tưởng "khoá key cho mod" (phần kinh tế)
Lập trường của tôi: **cho phép bản lậu như Minecraft ("just pirate it")**, không chạy theo chặn. Kèm một tính năng: **modder có thể chọn (tuỳ chọn) yêu cầu key bản quyền** để người chơi nạp mod của họ. Không phải chặn kỹ thuật, vì tôi chấp nhận từ đầu rằng crack rất dễ. Mục tiêu là dựng **một bức tường không có cửa** đánh vào tâm lý: khi người chơi bản lậu cố nạp một mod cần key, họ gặp thông báo *"người tạo mod này không muốn, và mình tôn trọng họ, nên bạn cần tắt mod để tiếp tục chơi"*, kèm biểu tượng chìa khoá trên trang tải mod.

Vì sao tôi nghĩ nó hiệu quả: người chơi lậu thường có câu chuyện đạo đức "chống nhà bán lẻ hút máu". Với cơ chế này họ đối đầu với **một modder cụ thể** mà họ có thể quý, nên lớp phòng vệ tâm lý bị bỏ đi. Modder lớn tâm huyết cũng thích có quyền kiểm soát. Tôi phân loại modder thành ba nhóm: ghét người chơi lậu, chấp nhận vì họ kiếm tiền từ mod, và trung tính.

**Claude hiểu nhầm lúc đầu** (tưởng là chặn khỏi nền tảng tải mod hoặc danh sách cấm từng người), tôi đính chính, rồi Claude phân tích lại:
- Câu chữ trên bức tường *chính là sản phẩm*: nên để modder tự viết một dòng, có ký tên, thay vì thông điệp của nền tảng.
- **Vì sao modder bật cờ này?** Theo phân loại ba nhóm của tôi, hai nhóm đầu không có lý do bật, vì bật là thu nhỏ khán giả. Cần "cà rốt": huy hiệu, vị trí nổi bật, quyền lợi thật.
- Cửa sổ hiệu quả là **sớm**, trước khi có công cụ bỏ qua key một cú nhấp.
- Người chơi bản quyền với một mod cần key có thể thành **người truyền bá**: muốn dùng mod này thì cần game gốc.

Ở lượt đầu, Claude còn nêu thêm các điểm sau (đọc kỹ lại mới thấy mình đã bỏ sót ở bản trước):
- **Gate bằng key chỉ là phễu chuyển đổi, không phải DRM.** Kiểm tra "có sở hữu game không" nằm phía máy người chơi nên bản lậu sửa thành `return true` được. Giá trị nằm ở tầng xã hội.
- **Nó bắt được "người chơi lậu biên"** (đã quý một modder, €20 không phải rào cản, quán tính mới là), một nhóm *nhỏ hơn* cách tôi diễn đạt. Claude dặn: *"đừng tự bán quá cho mình"*.
- **Phe mod "tự do" sẽ phản đối công khai** và có thể gọi nó là "DRM trá hình" → phải để cờ là **tuỳ chọn** và kèm quyền lợi thật cho modder dùng nó (nổi bật, chia doanh thu, ưu tiên trong trình duyệt mod).
- **Không cho modder lập danh sách chặn từng người** (dễ thành công cụ quấy rối, gánh hỗ trợ đổ về mình), chỉ cho điều kiện nhị phân "có sở hữu game / không". (Đây là cách Claude hiểu câu "cho ai đó hoặc không cho ai đó" của tôi; tôi đính chính rằng ý tôi chỉ là key hay không key.)
- **Sự dung túng bản lậu phải ngầm, không bao giờ công khai.** Nói "lậu cũng được" ở kênh chính thức sẽ làm các cửa hàng phân phối lo ngại và đâm vào đúng rủi ro pháp lý đang né; cùng kỷ luật với việc *không bao giờ nói chữ "Pokémon"*.
- **Cây gậy yếu, củ cà rốt mới là phần lớn tác dụng**: trình duyệt mod, tự cập nhật, chơi mạng, lưu đám mây.
- **Về thời điểm:** tung tính năng này ngay lúc ra mắt, khi chưa ai tin mình, rủi ro hơn nhiều so với khi đã có cộng đồng mod sôi động và vài modder lớn *chủ động xin*. *"Để nhu cầu kéo nó ra, đừng đẩy."*

## 6. Chiến lược đã dẫn tới việc làm gì (08–09/09)
Mình đối chiếu với bản ghi các ngày sau, từ tin nhắn của tôi:
- **08/09:** tôi chọn tập trung dựng *một* hệ trước, bàn đặc điểm của nhà và nội thất cộng vào vùng phòng, và quyết **độ trung thành của thú không do phòng quyết định** mà do hành động của người chơi (đưa đi khám, cho ăn ngon, cùng đi và thắng). Cùng ngày: nhân vật **ném "bóng" ra phía trước** bắt được thú (log ghi tỉ lệ 81%), **kho nhỏ 6 sinh vật**, panel Đội riêng (không tái chế UI tu tiên), kho lớn kiểu PC, chiêu 4 ô theo cấp.
- **09/09:** tôi muốn nạp dữ liệu Pokémon thật vào thư mục **dev** để dựng đúng cấu trúc và lối chơi, còn **bản public sẽ xoá**; bám mô hình tính sát thương của Showdown; đặt tên lớp **`CaptureTool`** chung để mod khác dùng, điều kiện áp ngay trong định nghĩa công cụ, gồm cả điều kiện môi trường; thiết kế lại lối điều khiển (WASD, chọn bằng chuột, `R` ném bóng để triệu hồi con đang được chọn, bắt thú bằng ô hotbar rồi bấm chuột, `B` menu, `P` PC, `Y` khoá camera, `E` tương tác, hotbar 8 ô).
- Đây chính là chiến lược "framework dùng chung cho mod" được chuyển thành code: các tên dùng chung (`CaptureTool`), dữ liệu có điều kiện, dữ liệu thật chỉ ở dev.

## 7. Những gì tôi chưa chốt / ghi trung thực
- Đây là **thảo luận chiến lược**, không phải kế hoạch đã chạy. Chưa có giá bán, mô hình doanh thu, hay ngày ra mắt trong bản ghi tôi đọc.
- Các nhận định về vụ kiện, bằng sáng chế, hay thị trường game sưu tập sinh vật là **kiến thức của Claude lúc đó**, tôi chưa kiểm chứng độc lập. Không nên coi là tư vấn pháp lý.
- Cơ chế key là ý tưởng; gate chỉ có tác dụng tâm lý. Claude cũng nhắc đây là **phễu chuyển đổi**, không phải DRM, và nhóm người chơi lậu "biên" bị tác động nhỏ hơn cách tôi diễn đạt.
- Vertical slice cho bản EA được Claude đề xuất: bắt vài con → mang về nhà → xem chúng nhận nghề, kết bạn, bị thương, hồi phục trong khi thị trấn tự chạy → có thể yêu một NPC.

## Điều tôi học được
Một chiến lược ra mắt cũng cần **thông số kỹ thuật làm điều kiện** (tầng mod phải sẵn sàng, từ vựng không trùng thương hiệu) và **điểm dừng** (khi nào greybox phải kết thúc). Ý tưởng hay vẫn cần người phản biện nêu chỗ gãy.

## Bằng chứng
[`docs/lessons/project_pokemon_pack_open_decisions.md`](../docs/lessons/project_pokemon_pack_open_decisions.md) · [`project_capture_tool.md`](../docs/lessons/project_capture_tool.md) · [`docs/MODDING_GUIDE.md`](../docs/MODDING_GUIDE.md) · bài [Modding](04-modding-va-modder.md)
