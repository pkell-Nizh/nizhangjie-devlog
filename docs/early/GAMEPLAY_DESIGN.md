# Nizhangjie — Tài Liệu Thiết Kế Gameplay

---

## 1. Tổng Quan Hệ Thống Chiến Đấu

Nizhangjie là game chiến thuật theo lượt kết hợp thời gian thực. Chiến đấu diễn ra real-time — nhân vật tự động tấn công liên tục — nhưng người chơi có thể **tạm dừng (Pause)** bất cứ lúc nào để ra lệnh chiêu thức, phối hợp đội hình, và lên kế hoạch chiến thuật.

Mục tiêu cốt lõi: **đọc trận địa → plan trong pause → execute chuỗi chiêu → phá trận địch**.

---

## 2. Cơ Chế Thời Gian Thực + Tạm Dừng

### Thời gian thực
- Tất cả nhân vật (ta và địch) liên tục **đánh thường** gây sát thương.
- Linh lực tích lũy dần theo từng đòn đánh thường.
- Nhân vật tự động nhắm mục tiêu gần nhất nếu không có lệnh cụ thể.

### Tạm dừng (Pause)
- Người chơi pause bất cứ lúc nào — thời gian đứng yên hoàn toàn.
- Trong pause, dựa vào **Tốc Độ (Speed)** của từng nhân vật, chỉ số **Hành Động (Action Gauge)** của họ sẽ được nạp.
- Nhân vật có Action Gauge đầy mới có thể kích hoạt chiêu thức trong lượt pause đó.
- Người chơi có thể ra lệnh cho **nhiều nhân vật trong cùng một lần pause** để phối hợp.
- Sau khi xác nhận, tất cả lệnh được execute theo thứ tự đã chọn khi unpause.

> **Tốc độ có giá trị thực sự**: tu sĩ tốc độ cao sẽ nạp Action Gauge nhanh hơn, tức là được hành động nhiều hơn trong cùng một khoảng thời gian chiến đấu.

---

## 3. Hệ Thống Linh Lực

### Nguồn hồi phục
| Nguồn | Tốc độ |
|-------|--------|
| Đánh thường trúng địch | Nhanh |
| Tự hồi khi không chiến đấu | Rất chậm (tránh trốn rồi tung chiêu) |
| Giết địch | Bonus tức thời |
| Kỹ năng bị động/chủ động hỗ trợ | Tùy chiêu |

### Sát Khí Chuỗi Giết (Kill Combo)
Khi giết nhiều địch liên tiếp trong một lần hành động:
```
Năng lượng bonus lần giết thứ N = 1 + 0.2 × N
```
Ví dụ: giết 5 địch liên tiếp → lần giết thứ 5 hồi gấp **2.0 lần** linh lực thông thường.

Cơ chế này thưởng cho người chơi chơi **tích cực và đúng chiến thuật**, không phải chơi ẩn núp.

---

## 4. Linh Căn & Nguyên Tố

### Linh Căn
Mỗi tu sĩ có một hoặc nhiều linh căn quyết định nguyên tố họ có thể sử dụng:

| Loại | Đặc điểm |
|------|----------|
| **Đơn Linh Căn** (Thiên Linh Căn) | Chỉ một nguyên tố, sát thương và hiệu ứng nguyên tố ×1.5, phản ứng nguyên tố mạnh hơn |
| **Song Linh Căn** | Hai nguyên tố, có thể tự tạo phản ứng, mỗi nguyên tố yếu hơn đơn |
| **Tạp Linh Căn** | Ba nguyên tố trở lên, linh hoạt nhất về chiến thuật, từng nguyên tố yếu nhất |

> Đơn linh căn không phải lúc nào cũng tốt hơn — họ cần đồng đội để tạo phản ứng nguyên tố.

### Các Nguyên Tố
Hỏa, Thủy, Lôi, Băng, Phong, Thổ (và có thể mở rộng thêm).

---

## 5. Phản Ứng Nguyên Tố

Khi hai nguyên tố chồng lên cùng một mục tiêu:

| Phản Ứng | Nguyên Tố | Hiệu Ứng |
|----------|-----------|----------|
| **Bốc Hơi** | Hỏa + Thủy | AOE damage burst |
| **Dẫn Điện** | Thủy + Lôi | Chain sang địch liền kề |
| **Bão Lửa** | Hỏa + Phong | DoT kéo dài, lan rộng |
| **Vỡ Băng** | Băng + Lôi | Physical damage bonus lớn |
| **Cát Bay** | Phong + Thổ | Giảm accuracy địch |

**Knockback transfer**: Khi địch bị đánh bay vào địch khác đang mang nguyên tố, trạng thái nguyên tố có thể lan sang — người chơi có thể **dùng địch như vũ khí** để tạo phản ứng dây chuyền.

---

## 6. Công Pháp & Chiêu Thức (Skill System)

### Bí Kíp (Skill Book)
Nhân vật học chiêu thức từ các **Bí Kíp** thuộc 8 hệ vũ khí:

> Kiếm Pháp · Thương Pháp · Đao Pháp · Đại Kiếm Pháp · Phiến Pháp · Ám Khí Pháp · Quyền Pháp · Cung Pháp

Mỗi bí kíp cung cấp một **bộ chiêu thức** gồm nhiều thẻ kỹ năng. Các chiêu trong cùng bộ có tính chất tương đồng và hỗ trợ lẫn nhau qua hiệu ứng và knockback.

### Thẻ Chiêu Thức (Skill Card)
Trong mỗi bộ công pháp có các thẻ chiêu thức chủ động:
- **Chiêu yếu**: Linh lực thấp, hiệu ứng nhỏ, dùng để setup hoặc mở đầu combo
- **Chiêu trung**: Linh lực vừa, sát thương và hiệu ứng ổn định
- **Chiêu mạnh**: Linh lực cao, sát thương lớn, hiệu ứng đặc biệt (knockback mạnh, AOE, phá thuẫn...)

Ngoài ra có **thẻ On-Hit**: áp dụng hiệu ứng lên N đòn đánh thường tiếp theo (ví dụ: "Áp Hỏa Khí — 3 linh lực, đòn đánh thường tiếp theo gây thêm Hỏa trong 5 cú").

### Pre-Set Combo Chain
Trước trận, người chơi đặt sẵn **2-3 chuỗi combo ưu tiên** cho từng nhân vật. Trong pause chỉ cần confirm hoặc override khi cần tình huống đặc biệt.

Combo chain mẫu:
```
[Áp Hỏa Khí - On Hit] → [Chấn Kích Cước - Knockback 4 ô] → [Liệt Hỏa Trảm - AOE Hỏa Burst]
```

### Menu Chọn Chiêu Trong Pause
```
Chọn nhân vật
  └─ Chọn bộ công pháp (ví dụ: Hỏa Linh Kiếm Pháp)
       └─ Chọn thẻ chiêu thức
            └─ Xác nhận → thêm vào queue
```
Linh lực còn lại hiển thị real-time khi chọn thẻ.

---

## 7. Hệ Thống Chấn Kích (Knockback)

### Knockback Tier
Knockback có tier tăng dần theo độ mạnh của chiêu:

| Tier | Đẩy được | Ví dụ chiêu |
|------|----------|-------------|
| 1 | Light (cung thủ, thám tử) | Đẩy cước cơ bản |
| 2 | Light + Medium | Chấn Cước Trung Cấp |
| 3 | Light + Medium + Heavy | Đại Chấn Cước — tốn nhiều linh lực |

Địch cấp Heavy kháng Tier 1-2 hoàn toàn. Cần Tier 3 hoặc phá Khí Thuẫn trước.

### Va Chạm Khi Knockback
| Kịch bản | Kết quả |
|----------|---------|
| Địch bay vào **tường** | Bonus damage, không tạo opening |
| Địch bay **xuyên qua đồng đội** của nó | Chain knockback, tạo opening, gây hỗn loạn đội hình |
| Địch bay vào đồng đội **mang nguyên tố** | Transfer nguyên tố, có thể trigger phản ứng |

### Ứng Dụng Chiến Thuật
```
Hàng trước địch dày, không thể xuyên qua
→ Chấn Kích Cước Tier 2 đẩy một địch ra khỏi hàng
→ Lộ ra một lỗ hổng
→ Tu sĩ tốc độ cao dash xuyên vào hàng sau
→ Tung AOE tiêu diệt hàng sau
```

---

## 8. Tầm Nhìn & Che Chắn (LOS & Cover)

| Loại Chiêu | Tầm Nhìn | Acc |
|------------|----------|-----|
| **Chiêu bay thẳng** (kiếm khí, lôi đạn) | Bị chặn bởi cover và địch | 100% |
| **Chiêu bay vòng cung** (tên lửa, đạn vòng) | Bypass cover | 70-80% |

Knockback là cách hiệu quả nhất để **buộc địch ra khỏi cover** mà không phải chịu penalty acc.

---

## 9. Khí Thuẫn & Phá Thuẫn (Shield & Break)

### Khí Thuẫn
Lớp bảo vệ năng lượng hình thành từ linh lực tu luyện. Có nhiều loại:

| Loại Thuẫn | Cơ Chế | Cách Phá |
|------------|--------|----------|
| **Chân Khí Giáp** | Giảm % sát thương nhận vào | Tập trung sát thương liên tục |
| **Phản Thuẫn** | Block hoàn toàn một số đòn, gây sát thương phản khi bị phá | Chiêu DoT hoặc chiêu xuyên giáp |
| **Nguyên Tố Thuẫn** | Kháng một nguyên tố cụ thể | Nguyên tố khắc chế |
| **Tinh Thần Giáp** | Tự hồi liên tục | Phải burst phá nhanh |

### Trạng Thái Lộ Thể (Exposed)
Khi Khí Thuẫn bị phá hoàn toàn:
- Nhận 100% sát thương (chỉ trừ giáp vật lý và kháng nguyên tố cơ bản)
- **Mất kháng knockback** — kể cả Heavy cũng có thể bị đánh bay
- Trạng thái kéo dài N giây (không trôi trong pause)

> **Chiến thuật then chốt**: Phải có action sẵn sàng để exploit Exposed ngay khi phá thuẫn — combo chain phải được plan trước.

### Synergy Phá Thuẫn + Knockback
```
Phá Khí Thuẫn của Iron Wall
→ Lộ Thể — mất kháng knockback
→ Tier 1 knockback cũng đủ đánh bay
→ Địch bay vào đồng đội → chain knockback
→ Phản ứng nguyên tố dây chuyền
```

---

## 10. Phân Loại Địch Nặng (Heavy Archetype)

Heavy không chỉ là "nhiều máu" — mỗi loại cần chiến thuật khác nhau:

| Archetype | Đặc Điểm | Counter |
|-----------|----------|---------|
| **Thiết Bích** | Kháng knockback, giảm sát thương % | Phá Khí Thuẫn trước → rồi knockback |
| **Kinh Thích** | Phản sát thương khi bị đánh trực tiếp | DoT, AOE gián tiếp, không đánh thẳng |
| **Hộ Pháp** | Taunt — buộc địch đánh vào nó | Bỏ qua, kill support phía sau trước |
| **Trầm Sa** | Làm chậm mọi thứ tiếp xúc | Knockback tầm xa, không tiếp cận |

---

## 11. Phối Hợp Đa Nhân Vật

Trong một lần pause, người chơi có thể ra lệnh cho nhiều nhân vật nếu họ đều có Action Gauge đủ.

Ví dụ phối hợp 3 nhân vật:

```
Nhân vật A (Song Linh Căn Hỏa-Phong):
  → Áp Hỏa Khí on-hit + Phong Chấn Trảm → Bão Lửa tự kích hoạt

Nhân vật B (Đơn Linh Căn Thủy):
  → Chấn Kích Cước đẩy địch mang Hỏa vào hàng sau
  → Địch mang Hỏa va vào địch hàng sau đang mang Thủy → Bốc Hơi

Nhân vật C (Striker tốc độ cao):
  → Dash xuyên qua lỗ hổng vừa tạo ra
  → Tung AOE vào hàng sau đang bị damage bởi Bốc Hơi
```

Kết quả: Hàng trước bị phá, hàng sau bị tiêu diệt, Kill Combo energy spike → có thể tiếp tục chuỗi action.

---

## 12. Vòng Lặp Chiến Đấu Hoàn Chỉnh

```
[Thời gian thực]
Nhân vật đánh thường → tích lũy Linh Lực + Action Gauge

[Pause]
Người chơi đọc trận địa:
  - Loại Khí Thuẫn địch?
  - Formation địch dùng?
  - Knockback mở lỗ hổng ở đâu?
  - Nguyên tố nào đang active?

Chọn chiêu cho từng nhân vật
Confirm → Execute

[Thực thi]
Combo chain diễn ra theo thứ tự
Kill Combo energy bonus nếu giết nhiều

[Lặp lại]
Unpause → thời gian thực tiếp tục
```

---

*Tài liệu này sẽ được cập nhật khi có thêm cơ chế mới.*
