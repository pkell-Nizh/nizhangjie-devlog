# XianXian — Phân Tích UI/UX Toàn Diện

> **Nguồn tham chiếu**: `XianXian_Gameplay_Design.md` · `GAMEPLAY_DESIGN.md` · `M4_README.md`
>
> **Game**: XianXian / Nizhangjie — Cultivation Management Sim (RimWorld-style, bối cảnh tu tiên)
> **Đồ họa**: 2D Top-down Grid (có thể render với góc nghiêng 2.5D Isometric)
> **Cơ chế điều khiển**: Point-and-click chính + Keyboard shortcut phụ
> **Hai chế độ chính**: Sim Mode (quản lý tông môn) & Combat Mode (RTwP grid-based)

---

## 1. Bố Cục HUD Tổng Thể

### 1.1. Sơ đồ vị trí các thành phần

```
┌─────────────────────────────────────────────────────────────────────┐
│  [LOGO/TITLE]     [⚡ TỐCĐỘ: ▶ ▶▶ ▶▶▶ ⏸]     [NĂM in-game] [DATE]  │ ← Topbar
├──────┬──────────────────────────────────────────────────────┬───────┤
│ 💎   │                                                      │       │
│12,450│                                                      │[MINI  │
│ 🌿   │              VIEWPORT CHÍNH                          │ MAP]  │
│  340 │         (Local Map Grid / World Map)                 │       │
│ ⛏   │                                                      │ [⚠️]  │
│   89 │                                                      │ EVENT │
│ 💊   │                                                      │ FEED  │
│   45 │                                                      │       │
│ 🔮   │                                                      │       │
│  72% │                                                      │       │
├──────┴──────────────────────────────────────────────────────┴───────┤
│  [👤 D]  [🏛 B]  [📋 Q]  [⚔️ F]  [🧙 C]    [TÔN MÔN ★★☆] [24/50]  │ ← Bottom bar
├─────────────────────────────────────────────────────────────────────┤
│  [TOOLTIP ZONE]                                                      │ ← Status bar
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2. Chi tiết từng vùng HUD (Sim Mode)

#### ▸ Topbar (cạnh trên — luôn hiển thị)

| Thành phần | Vị trí | Mô tả |
|---|---|---|
| **Tốc độ thời gian** | Giữa trên | Nút `⏸ ▶ ▶▶ ▶▶▶ ▶▶▶▶` — Pause / 1x / 2x / 4x / 8x |
| **Ngày / Năm in-game** | Phải của tốc độ | "Năm 3 · Tháng Giêng · Ngày 15" |
| **Menu chính** | Góc phải cùng | Hamburger icon `≡` → Save / Load / Settings / Quit |

**Lý do đặt Topbar**: Không gian trung tâm trên thuộc về thông tin thụ động (thời gian chạy). Người chơi không cần nhìn liên tục nhưng cần biết ngay khi glance.

#### ▸ Left Sidebar (cạnh trái — Resource Bar dọc)

Hiển thị 6 resource chính theo dạng compact: `[Icon]` trên `SốLượng` — **chỉ hiện số, không hiện rate**. Click vào bất kỳ resource nào → mở **Resource Detail Panel**.

```
💎        🌿        ⛏        💊        🔮        📜
12,450    340       89        45        72%       8
```

**Resource Detail Panel** (mở khi click):
- Số lượng hiện tại
- Rate thu/chi: `+35/ngày` từ các nguồn nào (Linh Điền, Quest, v.v.)
- Rate tiêu thụ: `-12/ngày` từ đâu (Lương đệ tử, Tụ Linh Trận upkeep, v.v.)
- Cảnh báo nếu sắp cạn (dự báo số ngày còn lại)

**Lý do đặt Left Sidebar cho Resources**: Resource là thông tin cần glance liên tục — đặt cạnh trái cố định giúp mắt không cần tìm. Không hiện rate inline để tránh rối mắt; muốn chi tiết thì click mở panel.

#### ▸ Right Sidebar (cạnh phải)

- **Minimap**: Hiển thị local map thu nhỏ, vị trí đệ tử (chấm màu), building (icon nhỏ), zone (màu nền). Click vào minimap → camera jump đến vị trí đó.
- **Event Feed**: Cuộn thông báo storyteller từ trên xuống. Màu sắc phân loại: 🟡 Vàng = cảnh báo, 🔴 Đỏ = khẩn cấp, ⚪ Trắng = thông tin. Click vào event → focus camera tới entity liên quan.

#### ▸ Bottom Bar (cạnh dưới — điều hướng panel + tông môn info)

Dải icon ngang, mỗi icon toggle panel tương ứng + thông tin sect cơ bản:

| Icon | Phím tắt | Panel mở ra |
|---|---|---|
| 👤 [D] | `D` | Disciple Panel (danh sách đệ tử) |
| 🏛 [B] | `B` | Building Panel (công trình) |
| 📋 [Q] | `Q` | Quest Board |
| ⚔️ [F] | `F` | Faction Diplomacy |
| 🧙 [C] | `C` | Character (avatar của player) |

Bên phải các icon: `[Tông Môn: Rank ★★☆]` · `[Đệ tử: 24/50]`

**Lý do đặt Bottom Bar cho panel icons**: Panel icon không cần nhìn liên tục — chỉ dùng khi cần mở menu. Đặt dưới cùng tránh cạnh tranh tầm nhìn với viewport và resource bar.

#### ▸ Status Bar (cực dưới)

**Tooltip Zone** — vùng text hiển thị thông tin chi tiết của bất cứ thứ gì đang được hover chuột.

---

### 1.3. HUD trong Combat Mode (RTwP)

Khi vào encounter combat, layout chuyển sang:

```
┌─────────────────────────────────────────────────────────────────────┐
│  [⏸ PAUSE]  [Lượt pause: 3]    COMBAT — Huyết Nguyệt Động         │
├───────────────────────────────────────────────────────────────────┤
│                                                                     │
│                    COMBAT GRID                                      │
│          (mỗi unit có HP bar nổi trên đầu)                         │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  [Unit A] HP ████░░ LL ██░░░   [Skill 1][Skill 2][Pass]  [Queue ▶] │
│  [Unit B] HP █████░ LL ████░   [Skill 1][Skill 2][Pass]  (waiting) │
│  [Unit C] HP ████░░ LL █░░░░   [Skill 1][Skill 2][Pass]  (gauge低) │
├─────────────────────────────────────────────────────────────────────┤
│  ENEMY: [Thiết Bích - Heavy] HP ██████  SHIELD: Chân Khí Giáp ██░  │
└─────────────────────────────────────────────────────────────────────┘
```

**Thành phần Combat HUD**:

| Thành phần | Mô tả |
|---|---|
| **HP Bar + LL Bar** | Mỗi friendly unit: HP (đỏ) + Linh Lực (xanh lam) |
| **Action Gauge** | Thanh nạp trong pause — đầy thì mới được dùng skill |
| **Skill Bar** | Hiện skill có thể dùng (dimmed nếu LL không đủ) |
| **Command Queue** | Hiển thị queue lệnh đã chọn trong pause hiện tại |
| **Enemy Info Panel** | HP + loại Shield + element đang active của địch được chọn |
| **Combo Chain Preview** | Khi hover skill → preview tổng damage + element reaction sẽ xảy ra |

---

## 2. Cơ Chế Tương Tác "Bấm Vào Đâu Để Làm Gì"

### 2.1. Sim Mode — Click trên Local Map Grid

#### Click chuột trái (Left Click)

| Đối tượng click | Hành động |
|---|---|
| **Ô trống (Grass/Stone)** | Hiện tile info tooltip (TileType, LinhKhí score, Zone hiện tại) |
| **Ô trống + đang ở Zone Painter mode** | Paint tile vào zone đang chọn |
| **Ô trống + đang đặt Building** | Đặt building xuống (nếu tile buildable + đủ resource) |
| **Building đã xây** | Mở Building Detail Panel |
| **Đệ tử (Disciple entity)** | Mở Disciple Detail Panel cho đệ tử đó |
| **Minimap** | Jump camera đến vị trí click |
| **Icon trong Bottom Bar** | Toggle panel tương ứng |

#### Click chuột phải (Right Click)

| Đối tượng click | Hành động |
|---|---|
| **Đệ tử đang được chọn** | Cancel selection / Deselect |
| **Ô trống khi đang đặt Building** | Cancel placement mode |
| **Bất cứ đâu trong panel** | Đóng panel hiện tại |

#### Kéo thả (Drag)

| Thao tác | Hành động |
|---|---|
| **Drag trên viewport** (chuột giữa hoặc phải + kéo) | Pan camera |
| **Drag item từ Inventory ra ô trang bị** | Equip item vào slot |
| **Drag đệ tử từ Disciple Panel lên Building icon** | Assign đệ tử vào building |
| **Drag building footprint lên grid** | Placement preview (màu xanh = OK, đỏ = blocked) |
| **Drag zone painter** | Fill vùng tiles liên tiếp vào zone |
| **Scroll wheel** | Zoom in/out viewport |

### 2.2. Click vào Đệ Tử — Luồng chi tiết

```
Click đệ tử trên map (hoặc trong Disciple List)
    │
    ▼
Disciple Detail Panel mở bên phải (slide in)
    ├─ Tab [Thông Tin]: Tên · Độ hiếm · Cảnh giới · Tuổi/Thọ
    ├─ Tab [Chỉ Số]: Core Stats 6 chỉ số · Combat Stats derived
    ├─ Tab [Traits]: 3-5 trait cards (màu theo loại)
    ├─ Tab [Trang Bị]: 9 slot inventory (click slot → mở item picker)
    ├─ Tab [Quan Hệ]: Fellowship với các đệ tử khác (graph nhỏ)
    └─ Tab [Công Việc]: Standing Order dropdown + Override assignment
         │
         ├─ [Standing Order] dropdown: "Tu luyện" / "Luyện đan" / "Khai khoáng" / ...
         └─ [Assign ngay] button → highlight các building phù hợp trên map
```

### 2.3. Click vào Building — Luồng chi tiết

```
Click building trên map
    │
    ▼
Building Detail Panel
    ├─ Tên building + Tier hiện tại (★★☆☆☆)
    ├─ [Workers]: Slot đệ tử đang làm việc (kéo thả để thay)
    ├─ [Production]: Input required → Output rate (có progress bar)
    ├─ [Upgrade]: Tier lên/xuống (hiện cost + requirement)
    └─ [Remove]: Phá dỡ building (confirm dialog)

Nếu building là Quest Board → mở Quest Board UI (xem section 3.4)
Nếu building là Market → mở Market UI
```

### 2.4. Click vào NPC / Entity trên World Map

```
World Map mode: click vào Settlement
    │
    ├─ Là thành chưa thăm → Tooltip: "Chưa khám phá — Di chuyển đến đây?"
    └─ Là thành đã biết → Settlement Panel:
         ├─ Thông tin: Tên · Faction chủ · Dân số · Prosperity
         ├─ [Dịch vụ]: Market / Inn / Quest Board (nếu có)
         ├─ [Ngoại giao]: Quan hệ hiện tại với tông môn player
         └─ [Dispatch đến đây]: Mở Caravan Outfitting UI
```

### 2.5. Context Menu — Khi nào dùng Context Menu thay vì Direct Action?

| Tình huống | Cách xử lý |
|---|---|
| Click đệ tử | **Direct → mở Detail Panel** (không context menu) vì action duy nhất là "xem info" |
| Click building | **Direct → mở Building Panel** |
| Click item trong Inventory | **Context Menu** xuất hiện tại vị trí click: [Trang Bị] [Bỏ] [Dùng] [Chia tách] |
| Click đệ tử trong **Combat Mode** | **Direct → hiện skill bar** của unit đó ở bottom HUD |
| Right-click bất cứ đâu | Đóng / Cancel |

**Quy tắc thiết kế**: Context menu chỉ xuất hiện khi có **≥ 2 action khả dụng** và action không thể đoán được từ context. Nếu chỉ có 1 action hợp lý → Direct action.

---

## 3. Luồng Thao Tác Menu Chính

### 3.1. Disciple Panel (Bảng Đệ Tử)

```
[D] hoặc click icon 👤 → Disciple Panel slide từ trái vào

Panel layout:
┌────────────────────────────────────────┐
│ 🔍 Search  [Sort: Cảnh Giới ▼] [Filter]│
├────────────────────────────────────────┤
│ [Trần Lập]  Trúc Cơ  ⭐⭐⭐  Loyal 85   │
│  └─ Standing: Luyện Đan  [Kỹ năng: 72%]│
│ [Lý Mộ Xuân] Luyện Khí 9 ⭐  Loyal 45  │
│  └─ Standing: Tu Luyện Đài [KN: 30%]   │
│ ...                                    │
├────────────────────────────────────────┤
│ [+ Chiêu Mộ Đệ Tử]   Tổng: 24/50     │
└────────────────────────────────────────┘

Click vào 1 đệ tử → Disciple Detail Panel (như 2.2)

[+ Chiêu Mộ] → Recruitment Panel:
  ├─ Hiện 3 candidate (random refresh có tốn linh thạch)
  ├─ Mỗi candidate: tên, độ hiếm (màu), core stats preview, 1-2 trait visible
  └─ [Chiêu Mộ: 150 💎] button → deduct resource + add to roster
```

**Overlay behavior**: Disciple Panel + Building Panel có thể mở đồng thời (side-by-side) để drag đệ tử từ panel trái vào building ở panel phải. Tối đa 2 panel cùng lúc — panel thứ 3 đẩy panel thứ nhất đóng lại.

### 3.2. Building Panel & Zone Painter

```
[B] → Building Panel

Tabs:
├─ [Danh Sách]: Hiện tất cả building đã xây, click → focus camera
├─ [Xây Mới]: Grid các loại building, click → Placement Mode
└─ [Zone]: Zone Painter mode

Placement Mode:
  → Cursor đổi thành ghost building (semi-transparent)
  → Valid tile: màu xanh lá  |  Invalid: màu đỏ
  → Click đặt xuống  |  Right-click hủy
  → Hiện cost preview khi hover

Zone Painter Mode:
  → Chọn loại zone (Cultivation / Farming / Mining / Living)
  → Drag trên tiles → paint zone (màu tint theo loại)
  → Ctrl+Z để undo vùng vừa paint
```

### 3.3. Inventory & Trang Bị Đệ Tử

```
Trong Disciple Detail Panel → Tab [Trang Bị]

Layout 9 slot hình người:
    [Mũ]
[Nhẫn] [Áo] [Ngọc bội]
[Găng]     [Giày]
    [Quần]
[Vũ Khí] [Pháp Bảo*]
          * chỉ Key Unit

Click vào slot trống → Item Picker mở:
  ├─ Hiện items trong sect inventory phù hợp với slot
  ├─ Sort theo Phẩm chất (màu sắc tier) / ATK / DEF
  ├─ Hover item → tooltip stat comparison (vs item đang mặc)
  └─ Click → Equip (nếu đủ điều kiện realm + element)

Click vào slot đã có item → Context Menu:
  ├─ [Xem chi tiết]
  ├─ [Tháo ra → Sect Inventory]
  └─ [Tháo ra → Đất]

Sect Inventory (kho chung):
  → Mở riêng qua building Đặc Cung (Treasure Vault)
  → Grid view, filter theo loại / tier
  → Drag & Drop item giữa đệ tử và kho
  → [Chia tách Stack]: slider chọn số lượng
```

### 3.4. Quest Board

```
Click vào building Quest Board trên map → Quest Board UI (full overlay)

┌──────────────────────────────────────────────────────┐
│ BẢNG NHIỆM VỤ — Bắc Cực Thành                       │
│ [Filter: Tất cả ▼] [Difficulty: ≤ 60]               │
├────────────┬─────────────────────────────────────────┤
│ Quest List │  Quest Detail (click để chọn)            │
│            │                                         │
│ ⚔️ Trừ Yêu  │  Tên: "Trừ 20 Yêu Thú Nhỏ"            │
│ 🚚 Giao Hàng│  Loại: MonsterHunt | Difficulty: 30    │
│ 🏔️ Bí Cảnh │  Địa điểm: Thảo Nguyên Bắc             │
│ ...        │  Thời hạn: 14 ngày | Dispatch: 5 ngày  │
│            │  Phần thưởng: 500💎 + 5 Yêu Đan        │
│            │                                         │
│            │  [Chọn Đệ Tử Phái Đi]                  │
│            │   Slot 1: [Trần Lập ▼]                  │
│            │   Slot 2: [Chọn... ▼]                   │
│            │  Power dự kiến: 85 vs 30 ✅             │
│            │                                         │
│            │  [✔ Nhận Nhiệm Vụ]  [✖ Bỏ Qua]        │
└────────────┴─────────────────────────────────────────┘

Sau khi nhận:
  → Quest hiện trong [Quest Log] (tab riêng ở Quest Board)
  → Đệ tử được phái → status = "Dispatched" (không làm việc khác)
  → Timer đếm ngược hiển thị trong Disciple Card
```

### 3.5. World Map

```
Click vào Minimap (khi đang ở chế độ Local Map) → World Map mở fullscreen

World Map UI:
┌──────────────────────────────────────────────────────┐
│  [← Về Local Map]    WORLD MAP    [Scan Linh Khí 🔮] │
│                                                      │
│   [Grid 300-500 tile, có thể zoom/pan]               │
│   - Thành lớn: icon ⭐ lớn                           │
│   - Thành nhỏ: icon ○ nhỏ                            │
│   - Tông môn player: icon 🏯                         │
│   - Bí cảnh: icon 🗝                                 │
│   - Vùng hoang: màu tối                              │
│                                                      │
│  Hover tile → Tooltip: LinhKhí score (nếu đã scan)  │
│  Click Settlement → Settlement Panel (xem 2.4)       │
└──────────────────────────────────────────────────────┘

[Scan Linh Khí] button: chỉ hiện nếu player có La Bàn Linh Khí
  / đủ cảnh giới / có Trait Thiên Linh Mục
  → Overlay màu gradient (xanh cao → đỏ thấp) trên map
  → Hover tile → hiện số cụ thể
```

### 3.6. Caravan Outfitting UI (trước chuyến đi)

```
Trigger: click [Dispatch đến X] trên Settlement Panel

┌──────────────────────────────────────────────────────┐
│  CHUẨN BỊ HÀNH TRÌNH → Bắc Cực Thành               │
│  Thời gian: 3 ngày | Rủi ro: ⚠️ Trung bình           │
├──────────────────────────────────────────────────────┤
│  Đệ Tử Đi Cùng (chọn tối đa 5):                    │
│  [Trần Lập ⭐⭐⭐] [Lý Mộ Xuân ⭐] [+ Thêm]          │
├──────────────────────────────────────────────────────┤
│  Đồ Mang Theo (limited carry weight):               │
│  Sect Inventory → Drag vào ô Caravan Slots (8 slot) │
│  [100 Đan Dược Thường] [3 Bùa Tấn Công] [...]       │
├──────────────────────────────────────────────────────┤
│  Preview: Power dự kiến vs. Rủi ro khu vực           │
│  ⚔️ Combat Power: 450 | ⚠️ Threat Estimate: 200     │
│  [Chia 2 đoàn] [✔ Khởi Hành] [✖ Hủy]             │
└──────────────────────────────────────────────────────┘
```

### 3.7. Character Creation (đầu run)

```
New Game → Character Creation (fullscreen, không thể skip)

Tabs theo thứ tự:
1. [Ngoại Hình]: Avatar appearance (static illustration, chọn từ preset)
2. [Linh Căn]: Chọn Linh Căn (Phong/Thủy/Hỏa/Thổ/Lôi/Băng/Kim/Mộc)
              → Ảnh hưởng combat style và công pháp khả dụng
3. [Traits]: Chọn trait bẩm sinh theo hệ thống **Point Buy**
              → Mỗi run có tổng điểm trait (tăng thêm khi mở Achievement)
              → Trait thường: tốn ít điểm | Trait mạnh: tốn nhiều điểm
              → Trait mạnh (vd: Thiên Linh Căn, Kiếm Thánh) chỉ xuất hiện
                trong pool nếu đã unlock qua thành tựu tương ứng
              → Không giới hạn số trait, chỉ giới hạn tổng điểm
4. [Bonus] (nếu có Achievement unlock):
              → Chọn bonus stat (từ achievement)
              → Chọn hack item mang theo (tối đa 2)
5. [Knowledge]: Hiện danh sách Knowledge Flag đã unlock từ các run trước
              → **Không cần chọn để mang theo** — đây là kiến thức của người chơi,
                không phải buff nhân vật
              → Khi gặp lại event tương ứng trong run mới, game tự động
                thêm option đặc biệt vào dialogue (không cần làm gì thêm)
              → Tab này chỉ để player xem lại những gì đã biết trước khi bắt đầu
6. [Confirm]: Summary + tên nhân vật → Bắt đầu

Navigation: [← Quay lại] [Tiếp theo →] ở đáy mỗi tab
```

### 3.8. Combat Pause Menu (RTwP)

```
Thời gian thực → Player nhấn [Space] → PAUSE

Trong pause:
  → Tất cả unit đứng yên
  → Action Gauge đếm ngược cho unit đủ điều kiện (thanh countdown)
  → Unit có gauge đầy: viền sáng lên → click để chọn

--- CƠ CHẾ SKILL SYSTEM ---

Đánh thường:
  → Gây sát thương ATK / Qi / bonus sát thương từ vũ khí
  → Có thể áp On-hit effect (tùy vũ khí + buff đang active)
  → Mỗi đòn đánh thường hồi năng lượng cho nhân vật

3 loại bộ skill (học từ công pháp tại base):
  ├─ Binh Pháp: gắn với vũ khí (Kiếm / Đao / Chùy / Đại Vũ Khí /
  │             Cung / Quạt / Bút Pháp / Ám Khí)
  ├─ Thân Pháp: kỹ năng di chuyển (lướt, dash, reposition)
  └─ Công Pháp: kỹ năng nguyên tố theo Linh Căn
               (Ngũ hành: Phong/Thủy/Hỏa/Thổ/Kim
                Biến dị: Lôi/Băng/Mộc/...)

Mỗi bộ skill có thể có hoặc không có Passive Skill đi kèm.

Một nhân vật có thể đeo nhiều bộ skill tùy thuộc tính (Linh Căn, vũ khí).
Các skill trong cùng bộ hỗ trợ, combo và tương tác với nhau.
Một số skill có thể khắc hoặc vô hiệu hóa skill của bộ khác (khắc chế).

Skill effects có thể bao gồm:
  • Áp hiệu ứng (Burn, Freeze, Stun, Slow…)
  • Áp ấn nguyên tố lên địch / vùng
  • Giảm chỉ số địch (DEF down, SPD down)
  • Lướt / Dash (Thân Pháp)
  • AOE — vùng ảnh hưởng nhiều tile
  • Đánh xa (projectile, không cần tiếp cận)
  • Tăng thủ / tạo shield
  • Phá Khí Thuẫn hiệu quả hơn thường
  • Tạo buff lên bản thân / đồng đội
  • Tạo buff áp lên hiệu ứng đòn đánh (on-hit buff)

--- SETUP SKILL TRƯỚC TRẬN (tại base) ---

Player vào Tàng Kinh Các / khu vực chuẩn bị → mở Skill Setup cho từng nhân vật:

┌──────────────────────────────────────────────────────┐
│  TRẦN LẬP — Skill Setup                             │
├──────────────────────────────────────────────────────┤
│  Bộ đang đeo: [Hỏa Linh Kiếm Pháp ▼] [Thân Pháp ▼]│
│                                                      │
│  Skill Pool (từ bộ đã đeo):                         │
│  [Áp Hỏa 3LL] [Liệt Hỏa 8LL] [Thiên Hỏa 15LL]    │
│  [Chú Hỏa On-Hit 3LL] [Lướt Kiếm 2LL] ...          │
│                                                      │
│  Skill Queue (kéo thả vào đây — thứ tự = ưu tiên): │
│  1. [Chú Hỏa On-Hit 3LL]                           │
│  2. [Liệt Hỏa 8LL]                                 │
│  3. [Thiên Hỏa 15LL]                               │
│  (nhân vật sẽ dùng tuần tự, lặp lại khi hết queue) │
│                                                      │
│  [Đổi bộ skill ▶]  [Lưu Setup]                     │
└──────────────────────────────────────────────────────┘

Nhân vật sẽ liên tục thực thi skill theo thứ tự queue cho đến khi
kết thúc action. Nếu không đủ LL cho skill tiếp theo → bỏ qua skill đó
hoặc dùng đánh thường để hồi LL.

--- TRONG COMBAT PAUSE ---

Click unit có gauge đầy:
  ┌──────────────────────────────────────────────────────┐
  │  TRẦN LẬP  LL: ██████░░ (75%)  Gauge: ████ (FULL)  │
  ├──────────────────────────────────────────────────────┤
  │  Skill Queue hiện tại:                              │
  │  1. [Chú Hỏa On-Hit 3LL] ✅                        │
  │  2. [Liệt Hỏa 8LL]       ✅                        │
  │  3. [Thiên Hỏa 15LL]     ⚠️ LL không đủ           │
  │                                                      │
  │  [Giữ Queue]  [Chỉnh sửa Queue ▶]  [Đổi bộ skill ▶]│
  │  [Đứng Yên / Pass]                                  │
  └──────────────────────────────────────────────────────┘

[Chỉnh sửa Queue]: Mở lại Skill Setup thu gọn để reorder trong pause
[Đổi bộ skill]: Switch sang bộ skill khác đang đeo (nếu có)

Chọn skill có target:
  → Một số skill cần chọn target (click vào địch / ô grid)
  → Hitbox preview hiển thị (3x3, line, cone, cross)
  → Confirm → skill thêm vào Command Queue

Command Queue (hiện ở bottom):
  [Unit A: Áp Hỏa Khí → Trần Lập → Chấn Kích Cước]
  [Unit B: Thủy Lưu Trảm]
  [Unpause ▶]

[Unpause ▶] hoặc [Space] → Execute tất cả lệnh theo thứ tự
```

### 3.9. Overlay Priority — Cửa Sổ Chồng Nhau

| Quy tắc | Mô tả |
|---|---|
| **Tối đa 2 panel đồng thời** | Panel 1 (trái) + Panel 2 (phải). Panel thứ 3 push panel 1 đóng. |
| **Fullscreen override** | World Map, Character Creation, Quest Board fullscreen → đóng tất cả panel đang mở |
| **Combat Mode** | Override tất cả — chỉ Combat HUD hiển thị |
| **Escape key** | Đóng panel đang active nhất (LIFO) |
| **Drag between panels** | Nếu 2 panel cùng mở, cho phép drag item / đệ tử giữa 2 panel |

---

## 4. Phản Hồi Tương Tác (Feedback Mechanism)

### 4.1. Visual Feedback khi Click

| Hành động | Visual Feedback |
|---|---|
| Click tile hợp lệ (Sim) | Ripple animation nhỏ tại điểm click + tile tint nhẹ 0.3s |
| Hover tile trên grid | Tile highlight outline (trắng) — đổi màu theo buildable/not |
| Building placement valid | Ghost building màu xanh lá (50% opacity) |
| Building placement invalid | Ghost building màu đỏ (50% opacity) + shake nhẹ khi thử click |
| Đặt building thành công | Building "drop" animation + dust particle |
| Hover đệ tử | Đệ tử highlight ring + cursor đổi thành pointer |
| Click đệ tử | Đệ tử "bob" animation nhỏ |
| Drag đệ tử → Building | Building glow pulse khi đệ tử đang được hover lên nó |
| Skill selection trong Combat | Hitbox area flash preview (màu cam) + AOE outline |
| Execute skill | Flash trắng, particle effect, screen micro-shake (nếu AOE lớn) |

### 4.2. Audio Feedback (UI Layer)

| Hành động | Âm thanh |
|---|---|
| Mở panel | Soft whoosh + paper/scroll unfurl sound |
| Đóng panel | Reverse whoosh nhẹ |
| Click button | Soft click (như gỗ bamboo nhẹ) |
| Confirm action quan trọng | Deep tone + subtle chime |
| Error / Invalid action | Low thud + brief shake |
| Nhận quest | Chime nhẹ + scroll sound |
| Đột phá cảnh giới thành công | Epic SFX + screen flash |
| Event tiêu cực (đệ tử bỏ đi) | Dramatic chord |
| Pause combat | Time-stop sound (reverb out) |
| Unpause combat | Time-resume sound (reverb in) |

### 4.3. Thông Báo & Alert System

**3 cấp độ thông báo**:

```
🟢 Toast (auto-dismiss 3s) — góc trên phải
   "Đan Dược +12 — Đan Phòng đã hoàn thành mẻ luyện"

⚠️ Persistent Notification — Event Feed (phải)
   "Trần Lập: Loyalty giảm xuống 45 — cần chú ý!"
   (tồn tại đến khi player click acknowledge)

🔴 Modal Alert — center screen (require dismiss)
   "Tẩu Hỏa Nhập Ma! [Đệ tử X] đang trong nguy hiểm"
   [Xử lý ngay] [Bỏ qua]
```

### 4.4. Tooltip System

- **Delay**: 400ms hover → tooltip xuất hiện (tránh tooltip spam khi di chuyển nhanh)
- **Smart position**: Tooltip luôn render trong viewport, tự đổi góc nếu gần edge
- **Tooltip có 3 cấp độ**:
  1. **Brief** (400ms): Tên + icon
  2. **Standard** (hover thêm 1s): Tên + stats chính + short description
  3. **Detailed** (Alt + hover): Toàn bộ info, công thức, source

---

## 5. Các Trường Hợp Ngoại Lệ & Lưu Ý UX

### 5.1. Click vào Vùng Không Hợp Lệ

| Tình huống | Xử lý |
|---|---|
| Click vào Water/Mountain tile | Tooltip brief: "Không thể xây dựng — địa hình không phù hợp". Không action nào xảy ra. |
| Click đặt building khi không đủ resource | Ghost building flash đỏ + tooltip resource còn thiếu (highlight resource đang thiếu ở left sidebar) |
| Click đặt building khi tile bị chiếm | Ghost building flash đỏ + outline entity đang chiếm tile |
| Click đệ tử đang bị Dispatch | Mở Disciple Panel nhưng action bị greyed out: "Đang đi nhiệm vụ — trở về ngày X" |
| Drag item vào slot sai loại | Item "snap back" về vị trí cũ + brief shake |
| Chiêu mộ khi đầy roster | Button [Chiêu Mộ] disabled + tooltip: "Tối đa 50 đệ tử — cần xây thêm công trình mở rộng" |

### 5.2. Occlusion — Vật Thể Che Khuất Nhau

Do game dùng 2D top-down grid, occlusion xảy ra ít hơn isometric thuần, nhưng vẫn cần xử lý khi building cao che đệ tử.

| Vấn đề | Giải pháp |
|---|---|
| Building che khuất đệ tử bên dưới | Khi player hover lên building → building fadeout xuống 40% opacity để lộ entity phía dưới |
| Nhiều đệ tử trên cùng tile | Stack indicator (số "×3" trên tile) + click → mini popup list các entity, chọn đệ tử muốn interact |
| Entity nhỏ bị khuất sau building lớn | Entity luôn render một indicator dot ở edge của building che nó — click dot → select entity |
| Floating HP bar che nhau trong Combat | Khi hover unit → các HP bar khác giảm opacity 50%, HP bar của unit hover hiển thị đầy đủ |

### 5.3. Xử Lý Nhiều Đệ Tử Cùng Lúc (50+ đệ tử)

| Tình huống | Giải pháp |
|---|---|
| Tất cả đệ tử hiển thị trên map gây rối | Có nút toggle [Ẩn Entity Icons] — chỉ hiện khi hover vùng đó |
| Scrolling Disciple Panel 50+ người | Virtualized list (chỉ render items đang visible) + Sort/Filter |
| Salary day — 50 notification cùng lúc | Batch thành 1 notification: "Đã trả lương 24 đệ tử — 2 đệ tử không đủ tiền lương ⚠️" |
| Mass assignment (đổi standing order cho nhiều đệ tử) | Multi-select (Shift+click) trong Disciple Panel + bulk assign action |

### 5.4. Trạng Thái Edge Case trong Combat RTwP

| Tình huống | Xử lý |
|---|---|
| Pause nhưng không unit nào có gauge đầy | Message nhỏ: "Chưa có đơn vị sẵn sàng hành động — Unpause để tích lũy thêm" |
| Chọn skill nhưng địch target đã chết khi execute | Engine auto-retarget sang địch gần nhất + thông báo "Mục tiêu đã chết — tự động đổi target" |
| LL không đủ khi đến lượt execute | Skill bị skip + text noti: "[Unit] bỏ lượt — không đủ Linh Lực" |
| Heavy địch kháng knockback Tier thấp | Hiệu ứng "Kháng!" xuất hiện trên địch + số 0 hiển thị kiểu khác màu thay vì damage number |
| Thiên Kiếp trigger trong combat | Cutscene ngắn (2-3s) → combat tiếp tục với wave thiên lôi thêm vào grid |

### 5.5. Lưu Ý UX Quan Trọng — Phù Hợp Với Design Philosophy

| Design Principle | UX Implementation |
|---|---|
| **"Player Control > Automation"** | Không có nút "Auto-assign tất cả đệ tử". Standing order phải set thủ công từng người. |
| **"Quality-driven failure"** | Khi production thấp → tooltip chỉ rõ "Đệ tử này Ngộ Tính kém → cap skill thấp" thay vì nói lỗi hệ thống |
| **"Sharp distribution"** | Disciple card thiên tài có visual khác biệt rõ ràng (animation aura, màu richer) — không để miss |
| **"Storyteller drives"** | Event từ storyteller có animation "scroll mở ra" khác biệt với notification thường |
| **"No RNG-drama"** | Breakthrough outcome hiện xác suất trước khi confirm. Player tự quyết định có muốn push không. |
| **"Knowledge flag = access"** | Khi có knowledge flag, choice mới trong event được highlight nhẹ với icon 👁 (không too obvious để giữ immersion) |

### 5.6. Onboarding & First-time UX

```
Lần đầu mở panel mới → Overlay tutorial (dismiss-able):
  "Đây là Disciple Panel. Kéo đệ tử vào Building để assign công việc."
  [Hiểu rồi] [Đừng hiện nữa]

Khi resource lần đầu về 0:
  → Resource bar flashes red + toast: "Hết [Resource X] — cần [action Y]"

Khi đệ tử lần đầu bị sắp tẩu hỏa:
  → Modal Alert (không dismiss được cho đến khi đọc xong 3 giây)
  "Cảnh báo: [Đệ Tử] đang tích lũy Tâm Ma — assign vào Bế Quan Phòng hoặc satisfy Desire sớm"
```

---

## 6. Tổng Kết — Ma Trận Click Action

| Đối Tượng | Left Click | Right Click | Hover | Drag |
|---|---|---|---|---|
| Tile trống (Sim) | Xem tile info | — | Outline highlight | Zone paint |
| Building đã xây | Mở Building Panel | — | Tên + production rate | — |
| Đệ tử trên map | Mở Disciple Panel | Cancel select | Ring highlight | Assign đến Building |
| Item trong Inventory | Context Menu | Bỏ nhanh | Stat tooltip | Equip / Kéo sang kho |
| Quest trong list | Mở Quest Detail | — | Brief description | — |
| Settlement (World Map) | Mở Settlement Panel | — | Faction + LinhKhí | — |
| Combat unit (friendly) | Chọn → hiện skill bar | Deselect | HP + LL info | — |
| Combat unit (enemy) | Target highlight | — | HP + Shield + Element | — |
| Minimap | Jump camera | — | Vị trí label | — |
| Resource bar item | Mở resource detail | — | Thu/chi breakdown | — |
| Event Feed item | Focus camera tới entity | Dismiss event | Full text preview | — |

---

*Tài liệu này được soạn dựa trên XianXian_Gameplay_Design.md v1.0 và GAMEPLAY_DESIGN.md (Nizhangjie). Cập nhật khi có thay đổi thiết kế ở M5+ (Storyteller UI, World Map Travel, Cultivation panel).*
