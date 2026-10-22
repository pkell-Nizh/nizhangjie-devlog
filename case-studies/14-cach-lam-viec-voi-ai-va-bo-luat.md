# Case study 14 — Cách tôi làm việc cùng AI: bộ luật, bộ nhớ dự án và kỷ luật "xong = chơi được"

**Nói thẳng:** dự án được xây bằng cách **cộng tác với AI lập trình (Claude Code; trước đó còn một công cụ AI khác)**. Phần tôi làm: chọn hướng, chốt đánh đổi, đặt tiêu chuẩn "xong", đọc số đo, và quyết định đổi hướng khi thấy đi sai. Phần Claude làm: viết và viết lại code, chạy test, đo, ghi tài liệu. Tôi viết điều này vì nhà tuyển dụng nên biết *quy trình thật*, không phải một bức tranh "tự làm hết".

## 1) Bộ luật dự án — mỗi luật gắn với một bug thật
File [`docs/CODE_SYSTEM_RULE.md`](../docs/CODE_SYSTEM_RULE.md) (373 dòng). Ví dụ:
- **A1 JSON-first** — mọi data là JSON, không ScriptableObject (để mod được).
- **A2 Không singleton** — dùng ServiceLocator.
- **A4 Tất định** — không để thứ tự `Dictionary` quyết định kết quả; rng có seed.
- **A5 Lõi ⟂ content-pack** — luật "quan trọng nhất" sau M37 ([case 05](05-tach-loi-khoi-tu-tien.md)).
- **A6 Cấm fork đường ống — hãy tổng quát hoá nó** (Companion/Beast từng fork AI riêng và phải trả giá).
- **B1 Ngưỡng vào ≠ ngưỡng ra** (vùng chết, chống nhấp nháy trạng thái).
- **B8 Cấm quét toàn cục trong vòng lặp mỗi khung** — rút ra trực tiếp từ [case 01](01-lag-thanh-pho-bon-lan-doan-sai.md).
- **B9 Trạng thái mới → tự hỏi "có sống qua save/load không?"** — sau [M61](13-do-tin-cay-luu-tru-va-su-co.md).
- **B10 Sim là dữ liệu C# thuần, tick không đụng Unity** + test canh biên giới.
- **C1 "Code viết xong nhưng chưa gắn vào scene"** — lỗi xảy ra 3 lần.
- **C3 Đo trước khi tối ưu · C9 Bản Burst phải giống bản C# từng bit.**
- **D1 "XONG" = người chơi chạm được · D5 Sửa nguyên nhân, không sửa triệu chứng.**
- **Phần E: checklist trước khi nói "xong".**

## 2) "Xong" nghĩa là gì
Ngày 01/08/2026 Claude báo tính năng nội thất thành phố "xong" vì test xanh. Tôi hỏi lại: *"khoan, cậu đã thêm nó vào game để mình chơi thử chưa?"* — và hoá ra nội thất sinh ra nhưng **vô hình** (renderer chưa đọc dữ liệu), hàm cung cấp chỉ được gọi trong file test, và không có đường chuột phải nào trong scene. Đây là lần thứ ba dính đúng bẫy "viết xong nhưng chưa gắn vào scene". → Luật: *trước khi nói "xong", phải trả lời được "người chơi chạm được chưa?" bằng bằng chứng chạy thật (ảnh Play Mode / log).* Nếu mới có test xanh thì gọi đúng tên: **"đã code + test xanh, CHƯA verify Play Mode"**. Bạn sẽ thấy cụm này khắp ghi chú của dự án.

## 3) Bộ nhớ dự án để làm việc trên nhiều phiên và nhiều tài khoản
Dự án luân phiên giữa **hai tài khoản AI** (hết hạn mức thì đổi) trên cùng một máy. Vì AI không nhớ phiên trước, tôi đặt ra:
- Một **thư mục ghi chú** (200+ file, mỗi file một sự thật + một chỉ mục `MEMORY.md`) — đây chính là các tài liệu trong [`docs/lessons`](../docs/lessons).
- Quy ước **phân vai**: tài khoản này làm tính năng/nội dung và kiểm định modding, tài khoản kia làm hiệu năng — tránh hai bên cùng sửa một thứ.
- Lịch sử git với commit message rõ ràng là "bản ghi đáng tin nhất giữa hai bên".
- Mỗi bẫy được ghi lại ngay (PowerShell `-replace` phân biệt hoa thường, BOM UTF-8, MCP biên dịch trên main thread, Unity không tick khi mất focus...), để khỏi dính lại.

## 4) Những lần chúng tôi sai và cách sửa (nên đọc)
- **Bốn vòng tối ưu sai hệ** → luật "đo trước" ([case 01](01-lag-thanh-pho-bon-lan-doan-sai.md)).
- **Nêu rủi ro không tồn tại** (sắp xếp vẽ với vật cao khi dự án không có vật cao) → luật "kiểm tra có tồn tại chưa rồi mới nêu như rào cản" ([case 02](02-draw-call-tilemap.md)).
- **Quên rằng xianxia chỉ là một content pack** → ghi thành feedback "core sim trước xianxia" ([case 05](05-tach-loi-khoi-tu-tien.md)).
- **Test đỏ 1 ca suốt nhiều ngày mà không ai chạy lại** (id thiếu tiền tố) → thêm vào checklist chạy lại hồi quy.
- **Số đo giả**: đo khi map chưa nạp xong; Profiler thổi phồng; lần chạy lạnh có JIT; remote desktop làm chờ GPU. → mỗi cái được ghi thành quy tắc đo.

## 5) Thói quen trình bày kết quả trung thực
Trong ghi chú tôi luôn tách: **đã code / test xanh / đã verify Play Mode / chưa verify / chưa rõ nguyên nhân / nợ còn lại / không làm có chủ đích**. Ví dụ: "p95 xấu hơn sau sửa, chưa rõ nguyên nhân", "đường nứt chưa tái hiện được", "còn ~5 ms chưa giải thích", "RAM chưa phân rã theo nhóm".

## Công cụ
Unity 6 · C# · Burst/Jobs · Unity Profiler · MCP cho Unity (điều khiển Editor, đọc console, chạy test, build) · git · PowerShell/bash.
