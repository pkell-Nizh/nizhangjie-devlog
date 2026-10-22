# XianXian — Project Bible

> **Source of truth duy nhất** cho toàn bộ project: gameplay design + kiến trúc kỹ thuật + milestone roadmap + setup instructions + mọi quyết định đã chốt.
>
> **Trạng thái**: v1.0 — living document, update khi có quyết định mới.
>
> **Dùng để**: paste sang Cowork, reference khi code, pitch cho collaborator/community, onboard new contributor.

---

## Mục lục

### PHẦN I — GAMEPLAY DESIGN
1. [Vision & Pillar](#1-vision--pillar)
2. [Core gameplay loop](#2-core-gameplay-loop)
3. [Cốt truyện & meta narrative](#3-cốt-truyện--meta-narrative)
4. [World map](#4-world-map)
5. [Đệ tử (Disciples)](#5-đệ-tử-disciples)
6. [Sect management](#6-sect-management)
7. [Economy & quest](#7-economy--quest)
8. [Combat](#8-combat)
9. [Cảnh giới tu tiên](#9-cảnh-giới-tu-tiên)
10. [Meta layer & roguelike](#10-meta-layer--roguelike)
11. [UI/UX touchpoints](#11-uiux-touchpoints)
12. [Modding scope](#12-modding-scope)
13. [Out of scope v1.0](#13-out-of-scope-v10)
14. [Design philosophies (quy chiếu khi unsure)](#14-design-philosophies-quy-chiếu-khi-unsure)

### PHẦN II — KIẾN TRÚC KỸ THUẬT
15. [Tech stack](#15-tech-stack)
16. [Cấu trúc folder](#16-cấu-trúc-folder)
17. [Nguyên tắc kiến trúc](#17-nguyên-tắc-kiến-trúc)
18. [EventBus system](#18-eventbus-system)
19. [Data pipeline (JSON + Odin)](#19-data-pipeline-json--odin)
20. [Team vs Faction (architecture note)](#20-team-vs-faction-architecture-note)

### PHẦN III — MILESTONE ROADMAP
21. [Milestone overview](#21-milestone-overview)
22. [M1 — Foundation (DONE)](#22-m1--foundation-done)
23. [M2 — Data & Stats](#23-m2--data--stats)
24. [M3 — Combat refactor](#24-m3--combat-refactor)
25. [M4 — Sim skeleton](#25-m4--sim-skeleton)
26. [M5 — Cultivation systems](#26-m5--cultivation-systems)
27. [M6 — Meta layer](#27-m6--meta-layer)
28. [M7 — Modding hooks](#28-m7--modding-hooks)

### PHẦN IV — SETUP & OPS
29. [Setup instructions](#29-setup-instructions)
30. [Pitfall & lưu ý đã gặp](#30-pitfall--lưu-ý-đã-gặp)

---

## 1. Vision & Pillar

### 1.1. One-line pitch
> Game quản lý mô phỏng bối cảnh tu tiên — bắt đầu từ tán tu vô danh, xây dựng tông môn lớn, push đệ tử đột phá cảnh giới, sống sót qua faction war và thiên kiếp, mục tiêu cuối: thăng tiên.

### 1.2. Cảm hứng tham khảo
- **RimWorld** — grid 2D local map, storyteller-driven narrative, colony sim tick-based
- **House of Legacy** — thiết kế local map, generational legacy
- **Immortal Tales** — cultivation loop, tu tiên flavor

### 1.3. Pillar (4 cột thiết kế)

1. **Quy hoạch chủ động, không phụ thuộc RNG**
   Player kiểm soát rủi ro qua zoning, resource management, equipment selection. RNG có nhưng không phải nguồn drama chính.

2. **Quality-driven failure**
   Failure không đến từ supply chain (như RimWorld) hay throughput (như Factorio), mà từ chất lượng đệ tử. Đệ tử phế → tu mãi không lên → tốn lương vô ích → phải quyết định đầu tư hay loại bỏ.

3. **Storyteller-driven world**
   Thế giới sống qua event do "storyteller" trigger, không phải hardcoded plot. Modder có thể viết storyteller mới = recreate tu tiên truyện favorite của họ.

4. **Meta progression có ý nghĩa**
   Mỗi run kết thúc → thành tựu unlock + avatar trở thành legacy NPC + knowledge flag unlock choice mới. Không có "true ending" theo nghĩa thường.

---

## 2. Core gameplay loop

### 2.1. Bốn giai đoạn

```
┌─ Giai đoạn 1: Tán tu ───────┐    ┌─ Giai đoạn 2: Khai tông ─────┐
│  1-3 giờ gameplay           │    │  Chọn vị trí trên world map  │
│  Solo avatar + recruit ≤5   │ ─► │  Resource + tu vi + quest    │
│  Quest, mở tiệm, lang thang │    │  Mở local map vùng đất hoang │
└─────────────────────────────┘    └──────────────────────────────┘
              │                                  │
              │ (có thể skip nếu chọn làm        │
              │  đại gia kinh tế trước)          ▼
              │                    ┌─ Giai đoạn 3: Phát triển ────┐
              │                    │  Tuyển đệ tử, xây building   │
              └────────────────────┤  Push đột phá cảnh giới      │
                                   │  Sống qua faction war        │
                                   │  Quản lý desire, loyalty     │
                                   └──────────────────────────────┘
                                                  │
                                                  ▼
                                   ┌─ Giai đoạn 4: Thăng tiên ────┐
                                   │  Đạt cảnh giới cuối           │
                                   │  End run, meta unlock         │
                                   │  Avatar → Legacy NPC          │
                                   └──────────────────────────────┘
```

### 2.2. Chu kỳ gameplay điển hình (1 năm in-game)

1. **Đầu năm**: storyteller chạy faction simulation, quyết định event lớn cho năm (war? thiên kiếp? bí cảnh mở?)
2. **Trong năm**: player quản lý sect — assign đệ tử, build/upgrade, push đột phá, dispatch quest
3. **Sự kiện rải đều**: storyteller drip event nhỏ (đệ tử desire, NPC ghé thăm, random encounter)
4. **Cuối năm**: faction war resolve (nếu có), reputation update, age tick

### 2.3. Tốc độ thời gian
- 10 phút real-time = 1 ngày in-game
- 1 năm in-game ≈ 60 giờ real-time (nếu chơi liên tục, không pause)
- Có pause + tốc độ 1x/2x/4x/8x — quan trọng cho management sim

---

## 3. Cốt truyện & meta narrative

### 3.1. Storyteller-driven (RimWorld pattern)

Không có "main plot" hardcoded. Thay vào đó: **storyteller** là AI quyết định khi nào event gì xảy ra. Storyteller "đẩy" event vào game qua EventBus.

**4 storyteller v1.0 (suggested, chưa final)**:

| Storyteller | Tính cách | Phù hợp với |
|---|---|---|
| **Thiên Đạo** | Tension tăng dần theo thời gian. Càng để lâu càng nhiều kiếp nạn. | Player thích pressure tăng dần. RimWorld Cassandra-style. |
| **Nhân Quả** | Phản ứng theo hành động player. Làm ác → tà phái gây sự. Làm thiện → chính phái ưu ái nhưng có kẻ ganh. | Player muốn hành động có hậu quả rõ ràng. |
| **Hỗn Độn** | Random pure. Hôm nay yêu thú, mai mưa linh thạch. | Player thích chaos, replay value cao. RimWorld Randy-style. |
| **Tự Định** | Event ít, sandbox thuần. | Player thích tự tạo content, không thích bị interrupt. RimWorld Phoebe-style. |

Storyteller = JSON config + (optional) C# logic. **Modder có thể thêm storyteller mới** = recreate tác phẩm tu tiên favorite của họ.

### 3.2. Faction war ("ai thắng do hên xui")

Mỗi năm in-game, storyteller có thể trigger faction war:

1. Storyteller chọn 2 faction dựa trên relationship, power, recent event
2. Resolve bằng công thức:
   ```
   winner = (factionA.power × roll(0.7-1.3)) vs (factionB.power × roll(0.7-1.3))
   ```
3. Loser mất territory/resource, winner gain reputation
4. Event publish lên GlobalEventBus → notification cho player

**Player chỉ tham gia khi muốn**: nếu tông môn player can dự (gửi đệ tử, kết minh) → mở combat encounter thật. Còn không thì war diễn ra abstract, player chỉ thấy kết quả.

→ Vừa có thế giới sống, vừa không tốn AI simulate từng trận.

### 3.3. Thăng tiên (endgame)

**Lore mơ hồ, không có epilogue**: avatar đạt cảnh giới cuối → biến mất "lên thiên giới" → end run. Không có tiên giới content trong v1.0.

Như Odyssey-style của RimWorld: feature endgame được flag là content cho expansion tương lai, không phải v1.0.

### 3.4. Knowledge flag system (biết trước cốt truyện)

Achievement unlock → `KnowledgeFlag` được lưu trong `ProfileSave` (cross-run persistence).

**Cách dùng**:
- Khi tạo run mới, achievement screen show các flag đã unlock
- Player **chọn mang theo** (giới hạn 3-5 cái mỗi run) — vì mang hết sẽ làm game quá dễ
- Trong run, khi event "Bà Lý mời uống trà" trigger:
  - Không có flag `knew_madam_li_betrays`: 2 choice (Đồng ý / Từ chối)
  - Có flag: 3 choice (Đồng ý / Từ chối / **"Ta biết ngươi có ý đồ xấu"** — unlock từ knowledge)

**Implementation**: JSON choice có field `requiredKnowledge: ["knew_madam_li_betrays"]`. Engine show/hide choice dựa trên flag.

### 3.5. Player identity

**Player = avatar có quyền lực + quan sát**

- Avatar là entity thật trong sim, không phải "ghost"
- Avatar có thể chết → end run
- Camera unlock — player quan sát toàn bộ tông môn, không cần đứng cạnh sự kiện
- Avatar tự tu luyện, đột phá, có goal riêng (mạnh lên + đạt thăng tiên)
- Avatar có thể trùng sinh trong run (chọn reset khi căn cốt quá yếu)

---

## 4. World map

### 4.1. Quy mô

- **300-500 tile** ô vuông
- **Mixed handcrafted + procedural**:
  - **5-7 thành lớn** handcraft (lore + NPC quan trọng + đặc thù riêng)
  - **20-30 thành nhỏ** template procedural (5-6 template, randomize tên + đặc điểm)
  - Vùng hoang, làng mạc nhỏ, bí cảnh → procedural
- **6-10 faction** start (chính phái, tà phái, trung lập, yêu thú, etc.)

### 4.2. Dynamic faction-level

Faction có thể war, conquer, suy tàn — nhưng location vật lý không đổi (thành A vẫn ở đó, chỉ đổi chủ).

State track per faction:
- Power (military strength)
- Territory list (control những thành nào)
- Reputation với 5-9 faction khác
- Treasury, key NPC

State track per thành:
- Owner faction
- Population, prosperity level
- Buildings/services (quest board, market, inn, etc.)
- Local NPC

### 4.3. Travel system

**Cost**: Time + Resource + Risk

- **Time**: tốn ngày in-game theo distance
- **Resource**: tốn lương thực phàm (cho đệ tử Luyện Khí) + linh thạch nếu dùng truyền tống trận
- **Risk**: random encounter trên đường

**Scale theo cảnh giới**: đệ tử cảnh cao (bay được) → ít cost, ít risk, đi xa nhanh. Tạo incentive push đệ tử lên cảnh giới.

**Caravan Outfitting UI** — UI riêng trước mỗi chuyến đi:
- Show sect inventory hiện có
- Slot mang theo (limited — quyết định mang đồ chiến đấu hay để lại phòng thủ)
- Travel time + risk preview
- Confirm departure

### 4.4. Random encounter

Trên đường travel, có thể gặp:
- Yêu thú tấn công
- Cường đạo phục kích
- Tu sĩ khác (faction khác, có thể friendly/hostile)
- Sự kiện random (lão già ăn xin, đan dược thất lạc, etc. — storyteller-controlled)

**Choice khi gặp**:
- **Bỏ của chạy** (mất loot/cargo, đệ tử an toàn)
- **Đánh** (dùng tài nguyên/đồ player đã mang theo — đây là điểm Caravan Outfitting matter)

### 4.5. Bí cảnh (dungeon)

**v1.0**: bí cảnh đơn giản — 1 local map combat + reward. KHÔNG có dungeon nhiều tầng.

**v1.x**: có thể expand thành dungeon nhiều tầng nếu cộng đồng yêu cầu.

### 4.6. Resource zone

**Mixed source**:
- Một số tài nguyên hiếm chỉ có ở wild (mỏ linh thạch lớn, rừng dược liệu hiếm)
- Đa số tài nguyên có thể sản xuất tại tông môn
- Vài resource chỉ mua được từ chợ (recipe đặc biệt, drop từ yêu thú cụ thể)

### 4.7. Vùng linh khí (cực kỳ quan trọng)

**Linh khí ảnh hưởng lớn tới tu luyện** — vị trí tông môn quyết định một phần lớn outcome.

**Cách player biết linh khí score của tile** (gating qua 3 path):
1. **Mua vật phẩm** "La bàn linh khí" hoặc tương đương — show linh khí score
2. **Tu vi đủ trình** (vd Luyện Khí Hậu kỳ) — tự cảm được
3. **Traits gacha** khi tạo nhân vật — "Thiên Linh Mục" (mắt thấy linh khí) → unlock từ đầu

**Safety net khi chọn sai vị trí**:
- **Dời tông môn**: cho phép (đắt + tốn thời gian, nhưng được)
- **Tụ Linh Trận**: linh khí nhân tạo → boost **output building**, KHÔNG boost đệ tử tu luyện một mình bên ngoài building

→ Trade-off chặt: location tốt → đệ tử tu nhanh dù ngồi đâu cũng được. Location thường + Tụ Linh Trận → building work tốt nhưng đệ tử "tự tu" thì chậm.

---

## 5. Đệ tử (Disciples)

### 5.1. Design philosophy

**Stats-heavy Character-like với sharp distribution**:
- Số chiều stat nhiều (linh căn, căn cốt, ngộ tính, linh thức, thể chất, etc.)
- 6 categories trait per đệ tử
- Sharp distribution: đa số bình thường, vài đệ tử cực mạnh đáng đầu tư
- **Không** có thinking AI (như Sims) — đệ tử là logic entity
- Player tự nhiên hình thành "key disciples" qua việc nhận ra thiên phú — không cần hardcode tag

**Khác RimWorld**: một đệ tử Thiên Linh Căn = power 3-4x đệ tử Linh Căn thường. Asymmetric power là feature, không phải bug.

### 5.2. Trait system

**3-5 trait per đệ tử**, từ 6 categories:

| Category | Ví dụ | Notes |
|---|---|---|
| **(a) Tính cách** | Kiên trì, lười biếng, kiêu ngạo, khiêm tốn, đa nghi, trung thành | Ảnh hưởng mood, relationship |
| **(b) Thiên phú** | Thiên Linh Căn, Hỏa Linh Căn, Thần Thức, Thiên Nhãn | Power multiplier lớn |
| **(c) Tật xấu** | Nghiện rượu, ham sắc, sợ ma, dễ tẩu hỏa, tham lam | Trigger event tiêu cực |
| **(d) Background** | Hoàng tộc, mồ côi, hậu duệ tiên nhân, ma tộc trá hình | Story event hook |
| **(e) Physical** | Căn cốt (HP, defense scale), Ngộ tính (skill cap) | Stats quan trọng nhất |
| **(f) Story** | "Chứng kiến đại kiếp", "Thù sâu với faction X" | Reactive event |

**Trait change**: gain & lose qua:
- **Nguyền rủa** (negative trait gain)
- **Event đặc biệt** — player chọn đệ tử tham gia event → có thể gain trait tốt hoặc xấu
- **Lợi thế biết trước cốt truyện** — knowledge flag cho phép player chọn đúng đệ tử cho đúng event

**Không có**: mất chi, bệnh tật (như RimWorld). Drama đến từ trait change qua event, không phải injury.

**Stats như căn cốt, ngộ tính: gần như không thay đổi** (sinh ra là vậy). Player phải quyết định: đầu tư hay loại bỏ.

### 5.3. Mood, desire, fellowship, lifestyle

- **Mood**: trạng thái cảm xúc hiện tại
- **Desire**: ham muốn ngầm — nếu kìm nén lâu → tích tụ **tâm ma** → tăng risk tẩu hỏa nhập ma
- **Fellowship**: relationship với đệ tử khác (xem 5.5)
- **Lifestyle**: phong cách sống (tu khổ hạnh, tu tiêu dao, tu thực chiến, etc.) — ảnh hưởng stat growth rate

**Desire system khác goal**:
- Goal: "muốn mạnh lên" (đơn giản, mọi đệ tử có)
- Desire: cụ thể hơn, có thể là "muốn 1 pháp khí thượng phẩm", "muốn đột phá Trúc Cơ trong 10 năm", "muốn báo thù faction X"
- Player có thể satisfy desire (tốn resource) hoặc ignore (chấp nhận rủi ro tích tụ tâm ma)

### 5.4. Expectation system (goal-fail → negative event)

**Trigger chain ví dụ**:
1. Đệ tử có trait `Tham Lam` → expectation = "lương linh thạch cao"
2. Player trả thấp trong 6 tháng in-game → tích tụ `Resentment` score
3. Khi `Resentment` đạt threshold + có trigger (đệ tử rảnh + có kho không khóa) → event "Đệ tử trộm kho"
4. Choice: phát hiện ngay (combat/trục xuất) hoặc không phát hiện (resource mất + reputation drop)

**Trigger ví dụ khác**:
- Trait `Thù ghét X` + low fellowship với đệ tử có background `Faction X` → event "Đệ tử giết bạn cùng môn"
- Trait `Tham vọng` + bị bỏ qua khi promotion → event "Đệ tử rời tông môn theo faction địch"

**Implementation**: `ExpectationSystem` track expectation per trait. `EventTriggerCondition` per event tiêu cực.

### 5.5. Relationship & social

**Complex relationship**:
- Bạn / đối thủ / sư đồ / tình cảm / thù
- Mỗi cặp có fellowship score (-100 to +100)
- Score thay đổi qua interaction (cùng làm việc, cùng combat, event)

**Sư-đệ system**: **chỉ key disciples** (player promote thành trưởng lão/đại đệ tử) mới có quyền nhận đệ tử riêng. Đệ tử thường không có sư phụ — tu luyện dựa vào trait + tài nguyên + công trình.

→ Cần UI **Promotion Panel**: player chọn đệ tử thường → promote thành trưởng lão. Sau promote unlock:
- Quyền nhận đệ tử riêng
- Quyền veto vài quyết định player
- Bonus stat (vd: +10% công pháp truyền dạy)

### 5.6. Tình cảm & hôn nhân

**Có gameplay đầy đủ**:
- 2 đệ tử **phải like nhau trước** (fellowship score cao + tương thích trait)
- Player có thể "se duyên" nếu cả 2 like nhau
- **Player KHÔNG ép được** nếu 2 người không có tình cảm — khác CK3 arranged marriage
- Có con → con thừa hưởng linh căn (di truyền)
- Avatar player cũng tham gia được (đạo lữ)

### 5.7. Aging & lifespan

**Lifespan scaling theo cảnh giới** (xem section 9 cho bảng cụ thể):
- Luyện Khí: 100 năm
- Trúc Cơ: 300 năm
- Kim Đan: 1000 năm
- ... etc

**Pressure tu luyện rõ ràng**: đệ tử không đột phá → chết già → mất công đầu tư. Player phải push họ.

Player thấy được "tuổi thọ còn lại" cho mỗi đệ tử → quyết định push ai đột phá trước.

### 5.8. Exit từ tông môn

**Đệ tử có thể rời nếu loyalty thấp** (qua trigger ở 5.4).
**Chiêu mộ chéo (faction khác chiêu mộ đệ tử mình)**: **HOÃN cho v1.x**.

### 5.9. Trait pool & balancing

**Start lean, expand dần**:
- v1.0 launch: 20-30 traits total (mỗi category 3-5 traits)
- Post-launch: expand qua update + modding

**Modder-first**: traits = JSON entry. Modder dễ add → giảm pressure tự design hết.

**Test bench**: cần in-game tool spawn variant đệ tử để test trait combo. (Editor tool, chưa cần ship vào game)

---

## 6. Sect management

### 6.1. Building system

**~12 building chính** (focused, không bloat), mỗi cái 3-5 tier.

**Draft 12 building v1.0**:

| # | Building | Vai trò | I/O |
|---|---|---|---|
| 1 | **Linh Điền** (Spirit Field) | Trồng linh thảo | linh khí → linh thảo |
| 2 | **Đan Phòng** (Alchemy Room) | Luyện đan | linh thảo + recipe → đan dược |
| 3 | **Luyện Khí Phòng** (Refinement) | Rèn pháp khí | linh khoáng + recipe → pháp khí |
| 4 | **Phù Lục Đường** (Talisman Hall) | Vẽ bùa | linh thạch + recipe → bùa chú |
| 5 | **Tu Luyện Đài** (Cultivation Platform) | Boost tu luyện rate | linh khí ambient → đệ tử tu tiến |
| 6 | **Bế Quan Phòng** (Seclusion Chamber) | Đột phá cảnh giới (an toàn hơn) | thời gian + linh thạch → chance breakthrough |
| 7 | **Tàng Kinh Các** (Library) | Lưu trữ tâm pháp/recipe | — (capacity) |
| 8 | **Tụ Linh Trận** (Spirit Gathering Array) | Linh khí nhân tạo (boost building lân cận) | linh thạch upkeep |
| 9 | **Đặc Cung** (Treasure Vault) | Kho | — (capacity) |
| 10 | **Trưởng Lão Phòng** (Elder Quarters) | Required cho promote key disciples | — (cho phép promote) |
| 11 | **Hộ Sơn Đại Trận** (Sect Defense Array) | Phòng thủ khi tông môn bị tấn công | linh thạch upkeep |
| 12 | **Truyền Tống Trận** (Teleport Array) | Travel nhanh giữa tông môn & thành lớn | linh thạch per use |

### 6.2. Building tier & gating

**Tier 1 → Tier 5** mỗi building. Higher tier:
- Output cao hơn
- Capacity lớn hơn
- Recipe slot nhiều hơn
- **Yêu cầu cảnh giới cao hơn** để vận hành (gating tự nhiên)

Ví dụ:
- **Đan Phòng Tier 1**: ai cũng vận hành được, luyện được đan Luyện Khí
- **Đan Phòng Tier 3**: cần Trúc Cơ vận hành, luyện được đan Trúc Cơ
- **Đan Phòng Tier 5**: cần Kim Đan vận hành, luyện được đan Kim Đan

**Build vs operate**:
- Xây Tier cao: ai cũng xây được (nhưng tốn resource khủng)
- Vận hành Tier cao: cần đệ tử cảnh giới đủ

### 6.3. Production chain

**Đơn giản 1-2 bước** — design choice intentional, không phải lười.

```
Linh thảo + recipe + đệ tử (trong Đan Phòng) → Đan dược
Linh khoáng + recipe + đệ tử (trong Luyện Khí Phòng) → Pháp khí
```

**Lý do đơn giản**: "đệ tử mình làm mà không làm được là do nó phế". Failure design = quality-driven, không phải logistic-driven.

→ Player không spend time tối ưu pipeline. Spend time đánh giá đệ tử + dispatch đúng người vào đúng building.

### 6.4. Recipe system

**Có đầy đủ — mỗi đan có recipe cụ thể**.

Recipe là:
- **Collectible**: quest reward, loot từ bí cảnh, mua từ chợ
- **Lưu trữ**: ở Tàng Kinh Các (capacity limit theo tier)
- **Tradable**: có thể bán cho faction khác (high value)

→ Tạo gameplay loop: "muốn luyện đan X → cần recipe X → recipe X từ đâu → bí cảnh Y / quest Z" → motivation để player active explore.

### 6.5. Resource (5-8 core)

**Core resources v1.0**:

| # | Resource | Role | Source |
|---|---|---|---|
| 1 | **Linh Thạch** | Currency chính + tu luyện fuel | Khai thác mỏ, quest reward, sell item |
| 2 | **Linh Thảo** | Alchemy material | Linh Điền, hái wild |
| 3 | **Linh Khí** | Intangible, vùng đất | Vùng linh khí natural + Tụ Linh Trận |
| 4 | **Linh Khoáng** | Forging material | Khai thác mỏ wild |
| 5 | **Đan Dược** | Consumable | Đan Phòng output |
| 6 | **Pháp Khí / Pháp Bảo** | Equipment | Luyện Khí Phòng output, drop, loot |

**Additional (không nhất thiết core)**:
- Bùa chú (consumable combat)
- Skill book / công pháp / tâm pháp (lưu Tàng Kinh Các)
- Equipment khác (mũ, áo, giày)

**KHÔNG có lương thực phàm + ngủ**: đệ tử là pure logic entity (sim tu tiên thuần, không phải RimWorld có tu tiên).

**Chỉ lương linh thạch**: đệ tử nhận lương theo cảnh giới + trait → expectation từ trait quyết định mức kỳ vọng.

### 6.6. Work assignment

**Manual assign** + **Standing order**

- **Standing order per đệ tử**: 1 dòng default order ("Tu luyện ở Tu Luyện Đài", "Trồng cây ở Linh Điền 2"). Đệ tử làm theo default khi không có lệnh khác.
- **Manual override**: player có thể tạm thời đẩy đệ tử vào việc khác. Hết việc → quay về standing order.

→ Đây là middle ground giữa pure manual (đau khi 50+ đệ tử) và priority list (RimWorld - quá automated cho identity sim tu tiên).

**UI**: Disciple panel — mỗi đệ tử có 1 dropdown "Default activity". Override = drag-drop vào building.

### 6.7. Trait → output (Linear multiplier)

Trait ảnh hưởng output **tuyến tính**:
- `Đan Tài` trait → +50% output Đan Phòng
- `Hỏa Linh Căn` → +30% chế đan hỏa-element
- `Lười biếng` → -20% mọi output

→ Player nhìn vào trait là biết đệ tử phù hợp building nào. Không cần threshold "phải có trait X mới làm được" — tránh feel "đệ tử phế hoàn toàn vô dụng".

### 6.8. Skill experience (cap theo trait)

Skill tăng theo thời gian làm việc, **nhưng cap theo trait**:
- `Ngộ Tính Cực Phẩm`: skill cap 100
- `Ngộ Tính Thường`: skill cap 50
- `Ngộ Tính Kém`: skill cap 25

→ Đệ tử ngộ tính thấp tu mãi vẫn không bằng ngộ tính cao mới học. Phù hợp tu tiên trope.

→ Có rate tăng nhanh đầu, chậm dần khi gần cap (asymptotic).

---

## 7. Economy & Quest

### 7.1. Currency

**Chỉ Linh Thạch (single tier v1.0)**.

**Note**: tiered linh thạch (hạ/trung/thượng/cực phẩm) là tu tiên trope nhưng **bỏ ở v1.0** vì:
- Simplify UX cho player mới
- Có thể add ở v1.x nếu cần exchange rate gameplay

### 7.2. Market (Fixed price v1.0)

**Fixed price** — chợ luôn mua/bán theo giá cố định.

**Note**: supply-demand dynamic là feature tốt nhưng **hoãn v1.x** — focus implement core trước.

### 7.3. Tiệm (Shop)

**Both modes — phụ thuộc giai đoạn**:

**Giai đoạn tán tu**:
- Player có thể thuê chỗ ở thành NPC mở **tiệm nhỏ**
- Passive income: NPC walk-in mua hàng
- Cần đệ tử trông tiệm (1 slot)

**Giai đoạn tông môn**:
- Tông môn có thể có **phân điếm** (branch shop) ở thành lớn
- Larger scale: nhiều slot, nhiều loại hàng, reputation gain

### 7.4. Quest types (9 loại v1.0)

Quest types listed ở priority (top → bottom):

1. **Đưa hàng / giao thư** (logistics) — A → B
2. **Săn yêu thú** (combat hunt) — kill X yêu thú, lấy mẫu
3. **Hộ tống** (escort) — bảo vệ NPC từ A → B
4. **Thu thập** (gathering) — hái dược / đào khoáng / lấy item
5. **Bí cảnh exploration** — clear dungeon, lấy reward
6. **Tu luyện hộ pháp** — bảo vệ NPC đang bế quan
7. **Ám sát** — giết NPC cụ thể (morality grey, ảnh hưởng karma)
8. **Faction commission** — làm việc cho faction X, tăng reputation faction đó
9. **Học việc / dạy việc** — dạy hoặc học từ NPC (skill book reward)

**Sẽ có rất nhiều quest variants** — quest = JSON template, generated by storyteller.

### 7.5. Quest delivery

**Quest board ở mỗi thành**.

Player đến thành → mở quest board → xem list quest available.

**Không** có NPC chủ động tìm player v1.0 (simplify implementation).

**Special quest** (story-critical) sẽ có channel khác — như dialogue trực tiếp với NPC quan trọng. Storyteller-controlled.

### 7.6. Quest difficulty

**Mixed**:
- **Generic quest**: scale theo cảnh giới đệ tử player có (auto-scaling pool)
- **Special quest** (story-critical): fixed difficulty, cần đủ trình mới làm được

→ Generic luôn có việc làm cho đệ tử yếu. Special là milestone player chuẩn bị kỹ.

### 7.7. Giai đoạn tán tu

**Duration**: ngắn (1-3 giờ gameplay), pacing nhanh, vào tông môn sớm.

**Optional path**: player có thể chọn làm **đại gia kinh tế** trước khi mở tông môn — kéo dài giai đoạn này nếu muốn (không buộc). Vd:
- Mở 3-5 tiệm ở các thành lớn
- Tích lũy tài sản 100k+ linh thạch
- Sau đó mở tông môn với điều kiện kinh tế khủng từ đầu

**Player có gì trong tán tu**:
- Avatar
- Có thể **recruit đệ tử sớm** (limit 3-5 trước khi khai tông)
- Inventory cá nhân
- Cửa tiệm nhỏ (optional)

**Khai tông unlock condition**:
- **Cả 2** — resource gate + quest reveal
- Resource: 10k linh thạch + cảnh giới Trúc Cơ (vd, balancing sau)
- Quest: "Khai Tông Quest" do storyteller trigger → reveal vị trí tốt + cho phép khai tông

---

## 8. Combat

**(Đã chốt ở phần trước, không có thay đổi từ thảo luận gameplay)**

### 8.1. Format
- **Grid-based 2D** — ô vuông
- Sub-system độc lập với management sim (combat encounter)
- Khi vào trận: sim pause hoặc tick chậm

### 8.2. Skill hitbox
- Đa hình dạng: 3x3, line, cone, cross
- Mở rộng được qua `IHitShape` interface (modder thêm shape mới)

### 8.3. Damage
- Công thức từ JSON (linh hoạt, mod được)
- Linear damage formula với crit, defense, element multiplier

### 8.4. Status effect & terrain
- Buff/debuff với duration
- Phản ứng nguyên tố:
  - Lửa + dầu = cháy lan
  - Băng + nước = đóng băng
  - Etc.

### 8.5. Combat unit composition
- Đệ tử tông môn = CombatUnit (assigned Team trong trận)
- Faction giữ nguyên (di sản từ Disciple) cho hậu quả meta
- Team gán bởi encounter setup → damage check dùng Team

### 8.6. Tu tiên-specific combat
- **Thiên kiếp**: trigger khi đột phá cảnh giới cao (cần combat survival)
- **Tẩu hỏa nhập ma**: thất bại khi tu luyện ép buộc → có thể trigger giữa combat (debuff)
- **Nhân quả**: giết quá nhiều → karma event spawn enemy mạnh hơn

---

## 9. Cảnh giới tu tiên

**(Phần này chưa thảo luận chi tiết — draft v0, sẽ refine ở milestone A trong thảo luận tiếp theo)**

### 9.1. Draft cảnh giới (8 cảnh, có thể điều chỉnh)

| # | Cảnh giới | Lifespan | Stats scale (vs base) | Note |
|---|---|---|---|---|
| 0 | **Phàm Nhân** | 60-80 | 1x | Trước tu tiên |
| 1 | **Luyện Khí** (9 tầng) | 100-150 | 2x-5x | Entry tu tiên |
| 2 | **Trúc Cơ** | 200-300 | 10x | Mid-low |
| 3 | **Kim Đan** | 500-1000 | 30x | Mid |
| 4 | **Nguyên Anh** | 2000-3000 | 100x | Mid-high |
| 5 | **Hóa Thần** | 5000-10000 | 500x | High |
| 6 | **Luyện Hư** | 20000+ | 2000x | Very high |
| 7 | **Đại Thừa** | "Tiên" | 10000x | Endgame |
| 8 | **Phi Thăng** | (lên thiên giới) | — | Win condition, end run |

### 9.2. Sub-tier (cho cảnh giới thấp)

Luyện Khí có **9 tầng** (1-9). Trúc Cơ và Kim Đan có **Sơ kỳ / Trung kỳ / Hậu kỳ / Đại viên mãn**.

Mỗi sub-tier nhỏ:
- Stats scale +20-30%
- Lifespan +10-20%
- Unlock skill mới

### 9.3. Đột phá

**Yêu cầu**:
- Linh thạch (scale theo cảnh giới)
- Đan dược tương ứng cảnh giới (vd: Trúc Cơ Đan để break vào Trúc Cơ)
- Thời gian bế quan
- (High tier) Pháp khí tương ứng

**Risk**:
- Tẩu hỏa nhập ma (failure → debuff vĩnh viễn)
- Thiên kiếp (cao cảnh giới → trigger combat)

**Bế Quan Phòng** giảm risk + tăng success rate.

---

## 10. Meta layer & roguelike

### 10.1. Run-end conditions

- Avatar chết (combat, event, age out trước khi đột phá cảnh cao đủ)
- Avatar thăng tiên (đạt Phi Thăng)
- Player chủ động end run (giao avatar về dưỡng già, start new run)

### 10.2. Trùng sinh (in-run)

Khi avatar trong run thấy quá yếu (căn cốt thấp, trait xấu) → player chủ động trigger trùng sinh:
- Mất tiến trình hiện tại của avatar (level, một số kỹ năng)
- **Giữ một phần ký ức**: vài trait được unlock (từ thành tựu in-run), một phần stat meta, vài pháp bảo "ký vật"
- Avatar mới có **ID khác + link `previousLifeId`** — NPC khác (thiên cơ cao) có thể nhận ra
- Mở đường cho karma system

### 10.3. Trùng sinh (meta-run)

Khi run kết thúc:
- Avatar cũ → **Legacy NPC** trong run sau (có thể gặp lại, có thể là sư phụ hoặc địch)
- Achievement unlock → bonus stat / hack items / knowledge flag
- Player tạo new avatar:
  - Chọn bonus stat (từ achievement)
  - Chọn 1-2 "hack items" (vật phẩm mạnh đầu game)
  - Chọn 3-5 knowledge flag mang theo

### 10.4. Achievement examples (draft)

- **"Đan Đạo Tổ Sư"**: luyện 1000 đan trong 1 run → unlock bonus +20% Đan Tài trait pool
- **"Diệt Tà Trừ Ma"**: giết 100 ma đạo NPC → unlock hack item "Trảm Yêu Kiếm"
- **"Đào Lý Mãn Thiên Hạ"**: 50 đệ tử thăng tiên → unlock knowledge flag "knew_secret_dao_of_teaching"
- **"Đạo Lữ Trăm Năm"**: avatar có đạo lữ + cùng đột phá tới Kim Đan → unlock cosmetic

### 10.5. Legacy NPC

Avatar run trước → NPC trong run sau với:
- Tên, traits, achievement đã đạt
- Có thể là **sư phụ tiềm năng** cho avatar mới
- Có thể là **địch** (nếu avatar trước là tà phái)
- Có thể **trao lại pháp bảo / công pháp** đã có
- Storyteller có thể trigger event "Bạn gặp di sản của X" → dialogue đặc biệt

---

## 11. UI/UX touchpoints

Các UI panel quan trọng (cần thiết kế kỹ):

### 11.1. Disciple panel
- List tất cả đệ tử, sortable theo cảnh giới / trait / loyalty / age
- Detail view: trait, stats, mood, desire, fellowship, lifestyle
- Standing order dropdown
- Promote button (nếu đủ điều kiện)

### 11.2. Caravan Outfitting UI (trước mỗi chuyến đi)
- Show sect inventory hiện có
- Slot mang theo (limited)
- Travel time + risk preview
- Confirm departure
- "Chia thành 2 nhóm" option (đoàn lớn + đoàn hậu phương)

### 11.3. Linh khí scan UI
- Trên world map, hover tile → show linh khí score (nếu player đủ điều kiện scan)
- 3 cách unlock: vật phẩm la bàn / cảnh giới đủ trình / trait Thiên Linh Mục
- Trước khi chọn vị trí khai tông: show top 5 tile linh khí cao nhất player đã scan

### 11.4. Quest Board (tại mỗi thành)
- List quest available
- Filter theo type / difficulty / reward
- Detail view: requirement, reward, time limit, faction giver
- Accept → assign đệ tử

### 11.5. Faction Diplomacy panel
- Show 6-10 faction, relationship với player tông môn
- Track power, territory, recent event
- Action: kết minh, tuyên chiến, gửi sứ giả

### 11.6. Sect Overview Dashboard
- Tổng resource, đệ tử count, building count
- Recent events feed (storyteller-published)
- Tu luyện progress của key disciples
- Warning system (đệ tử sắp tẩu hỏa, kho gần đầy, etc.)

### 11.7. Character Creation (start of run)
- Choose avatar appearance (basic, không animation)
- Choose race / bloodline
- Choose initial trait pool
- Choose bonus stat (from achievement unlock)
- Choose hack items (from achievement unlock)
- Choose knowledge flags to bring (limit 3-5)

### 11.8. Promotion Panel (cho key disciples)
- List đệ tử đủ điều kiện
- Promote → unlock master-disciple system + bonus

---

## 12. Modding scope

### 12.1. Data modding (M7)
- Mọi JSON data có thể override:
  - Trait (add/modify/remove)
  - Building (add/modify)
  - Recipe (add/modify)
  - Quest template
  - Storyteller
  - Faction
  - Event
  - Region

### 12.2. Code modding (M7)
- BepInEx + Harmony
- Public ModApi surface (`Modding/ModApi.cs`)
- EventBus interceptor pattern — modder không cần Harmony patch để theo dõi event

### 12.3. Modder use cases mục tiêu

- **Recreate tu tiên truyện**: modder viết storyteller + faction + event package → game cảm giác như novel đó
- **Add cảnh giới mới**: thêm cảnh giới Tu Tiên Đại Đế trên Phi Thăng (DLC-style)
- **Race expansion**: thêm yêu tộc, thần tộc, ma tộc với race-specific traits

### 12.4. Asset modding
- Raw PNG drop-in (loader tự đăng ký fake address vào Addressables)
- Không buộc modder build Addressable bundle qua Unity

---

## 13. Out of scope v1.0

Thứ KHÔNG làm v1.0 (đẩy v1.x hoặc DLC):

### 13.1. "Play as disciple" mode
Player điều khiển 1 đệ tử thường, không phải sect manager. Modding framework có sẵn từ M7, nhưng UI/UX riêng = post-launch.

### 13.2. Tiên giới content
Thăng tiên = end run, không có epilogue. Tiên giới = expansion tương lai.

### 13.3. Bí cảnh nhiều tầng
v1.0: bí cảnh = 1 local map combat. Multi-floor dungeon = v1.x.

### 13.4. Supply-demand market
v1.0: fixed price. Dynamic market = v1.x.

### 13.5. Tiered linh thạch
v1.0: single tier currency. Hạ/trung/thượng/cực phẩm = v1.x nếu cần.

### 13.6. Chiêu mộ chéo (faction khác chiêu mộ đệ tử mình)
v1.0: chỉ có "đệ tử tự rời nếu loyalty thấp". Faction spy/chiêu mộ = v1.x.

### 13.7. NPC chủ động tìm player
v1.0: quest board only. NPC walk-in = v1.x.

### 13.8. Full economic sim faction-to-faction trade
v1.0: faction war + reputation only. Trade simulation = v1.x.

---

## 14. Design philosophies (quy chiếu khi unsure)

Khi gặp trade-off mới, dùng các nguyên tắc này:

### 14.1. "Quality-driven failure, not logistic-driven"
Failure đến từ đệ tử yếu, không phải supply chain gãy. Khi gặp choice "thêm logistic complexity?" → no.

### 14.2. "Sim tu tiên thuần, không phải RimWorld có tu tiên"
Đệ tử là logic entity, không có food/sleep/animation. Khi gặp choice "thêm need cho đệ tử?" → no.

### 14.3. "Sharp distribution, asymmetric power"
Đa số bình thường, vài cực mạnh. Khi gặp choice "balance để mọi đệ tử có ích?" → no, để asymmetric là feature.

### 14.4. "Storyteller drives, sim runs"
Game cảm giác sống do storyteller "DJ chọn bài" trên nền sim. Khi gặp choice "hardcode scripted event?" → ưu tiên storyteller-mediated.

### 14.5. "Modder-first cho content depth"
Số trait/event/storyteller v1.0 lean để launch nhanh. Expand qua modder. Khi gặp choice "tự viết 200 trait?" → no, viết 30 + framework cho modder.

### 14.6. "Knowledge flag = mechanical access, not deterministic"
Biết trước cốt truyện cho choice/option, không phải auto-win. Player vẫn cần skill để execute.

### 14.7. "Player control > automation"
Khi gặp choice "automate cho player đỡ click?" → ưu tiên manual + standing order, không auto-prioritize.

### 14.8. "No RNG-driven drama, but RNG-driven flavor"
Outcome lớn (đột phá, war) có RNG nhưng player kiểm soát đa số. RNG là flavor (event flavor text, encounter type), không phải gating.

---

## Appendix A: Quyết định đã chốt qua thảo luận

### H — Cốt truyện
- H1: Storyteller-driven (RimWorld pattern) + modder content
- H2: Đơn giản, không có tiên giới content v1.0
- H3: Player = avatar có quyền lực + quan sát
- H4: Knowledge flag = Mechanical access

### E — World
- E1.1: 300-500 tile mixed handcraft + procedural
- E1.2: 5-7 thành lớn handcraft + 20-30 thành nhỏ template
- E2.1: Faction-level dynamic
- E2.2: 6-10 faction
- E3.1: Time + Resource + Risk
- E3.2: Random + scripted storyteller
- E3.3: Bí cảnh đơn giản
- E4.1: Mixed wild + sect + market
- E4.2: Vùng linh khí ảnh hưởng lớn + 3 cách scan + dời + Tụ Linh Trận

### B — Đệ tử
- B0.1: Stats-heavy Character-like (sharp distribution)
- B1.1: 3-5 trait
- B1.2: 6 categories (a-f)
- B1.3: Trait gain/lose qua event, stats core không đổi
- B2.1: Goal đơn giản
- B2.2: Trigger event tiêu cực (trộm, phản, giết)
- B3.1: Relationship phức tạp
- B3.2: Key disciples = player chỉ định
- B3.3: Hôn nhân full gameplay + di truyền + 2 phải like nhau
- B4.1: Lifespan scaling theo cảnh giới
- B4.2: Có thể rời nếu loyalty thấp
- B4.3: Chiêu mộ chéo HOÃN v1.x

### D — Sect management
- D1.1: 10-15 building focused
- D1.2: 3-5 tier mỗi loại
- D1.3: Cao cấp yêu cầu cảnh giới
- D2.1: 1-2 bước (quality-driven)
- D2.2: Recipe đầy đủ
- D3.1: 5-8 resource
- D3.2: Linh thạch + thảo + khí + khoáng + đan + pháp khí + (additional)
- D3.3: Chỉ lương linh thạch
- D4.1: Manual + Standing order
- D4.2: Linear multiplier
- D4.3: Skill exp + cap theo trait

### F — Economy & Quest
- F1.1: Linh thạch single tier
- F1.2: Fixed price
- F1.3: Both - tán tu mở tiệm nhỏ + tông môn có phân điếm
- F2.1: 9 quest types
- F2.2: Quest board mỗi thành
- F2.3: Mixed - generic scale, special fixed
- F3.1: Ngắn 1-3h + optional kéo dài làm đại gia kinh tế
- F3.2: Avatar + recruit sớm (limit 3-5)
- F3.3: Resource + Quest

---

## Appendix B: Còn cần thảo luận (post-doc)

Các vùng chưa thảo luận hoặc cần refine:

- **Cảnh giới cụ thể** (section 9 draft) — refine bảng lifespan, stat scale, sub-tier
- **Combat feel detail** — turn-based vs RTwP, team size, line-of-sight, Qi cost vs cooldown
- **Tone & thẩm mỹ** — nghiêm túc vs châm biếm, mức độ dark content, ma đạo allowed
- **UX & onboarding** — tutorial, tooltip culture, pause-and-plan moments
- **Endgame & replayability** — run length, replay motivation, hand-crafted vs procedural ratio

Khi vào balancing phase + content creation, sẽ touch các vùng này.

---

*Doc v1.0 — chốt sau thảo luận H/E/B/D/F. Update khi có decision mới.*

---
---

# PHẦN II — KIẾN TRÚC KỸ THUẬT

---

## 15. Tech stack

| Thành phần | Quyết định | Note |
|---|---|---|
| Engine | Unity 6 (6000.x) | |
| Render Pipeline | **URP với 2D Renderer** | Built-in deprecated từ Unity 6.5 |
| Template khởi tạo | Universal 2D Core | |
| Ngôn ngữ | C# | |
| Input | Unity Input System mới | Active Input Handling = Both lúc đầu |
| Data source of truth | **JSON** | POCO + Odin Editor window, không SO |
| Asset bundling | Addressables | |
| Pathfinding | A* Pathfinding Project Free | Upgrade Pro khi cần (50+ đệ tử pathfind) |
| Tween/animation | DOTween free | |
| Camera | Cinemachine 3.x | API khác hoàn toàn so với 2.x |
| Editor productivity | Odin Inspector + Serializer (paid) | |
| Modding | BepInEx + Harmony (M7) | |

---

## 16. Cấu trúc folder

```
Assets/Scripts/
├── Core/                          # Engine layer - chạy từ đầu
│   ├── Bootstrap/GameBootstrap.cs
│   ├── Enums/Faction.cs           # meta-game
│   ├── Enums/Team.cs              # combat-only
│   ├── Events/                    # EventBus system (full M1)
│   │   ├── GameEvent.cs
│   │   ├── IEventBus.cs
│   │   ├── EventBus.cs
│   │   ├── ScopedBuses.cs         # Global / Combat / LocalMap
│   │   ├── EventListenerBehaviour.cs
│   │   ├── EventLogger.cs
│   │   ├── SampleEvents.cs
│   │   └── Editor/EventDebuggerWindow.cs
│   ├── Time/GameClock.cs          # Tick system (Fast/Slow/Day)
│   ├── Save/                      # (M6)
│   ├── ServiceLocator.cs
│   └── IsExternalInit.cs          # C# 9 shim
│
├── Data/                          # (M2) — JSON + Odin
│   ├── Loaders/                   # JsonDataLoader, DataRegistry
│   ├── Definitions/               # POCO: UnitDefinition, SkillDefinition, etc.
│   └── Editor/                    # Odin window edit JSON trực tiếp
│
├── Stats/                         # (M2) — horizontal system
│   ├── CharacterStats.cs
│   ├── StatModifier.cs
│   ├── ModifierSource.cs          # Equipment/Trait/Buff/Realm/Karma
│   └── Enums/StatType, ModifierType, CultivationRealm, ElementAffinity
│
├── Equipment/                     # (M2)
│   ├── EquipmentItem.cs
│   ├── Inventory.cs
│   └── Enums/EquipmentSlot, ItemRarity
│
├── Cultivation/                   # (M5) — horizontal system
│   ├── CultivationManager.cs      # Lên cảnh giới, breakthrough
│   ├── HeavenlyTribulation.cs     # Thiên kiếp
│   ├── QiDeviation.cs             # Tẩu hỏa nhập ma
│   ├── KarmaSystem.cs             # Nhân quả
│   └── Reincarnation.cs           # Trùng sinh (trong-run)
│
├── World/                         # (sau M5) — World map
│   ├── WorldMap.cs                # Grid ô vuông
│   ├── WorldTile.cs
│   ├── Region.cs
│   ├── Settlement.cs              # Thành NPC
│   └── WorldNavigation.cs
│
├── LocalMap/                      # (M4) — RimWorld-style grid
│   ├── Grid/                      # Tile, GridSystem
│   ├── Buildings/                 # Building + production
│   ├── Zoning/
│   └── Pathfinding/               # A* wrapper
│
├── Sim/                           # (M4) — Management sim
│   ├── Sect/                      # Sect-level aggregate
│   │   ├── Sect.cs
│   │   ├── SectResources.cs
│   │   └── SectReputation.cs
│   ├── Disciples/                 # Disciple data + assignment
│   │   ├── Disciple.cs
│   │   ├── DiscipleRegistry.cs
│   │   ├── Recruitment.cs
│   │   └── Traits/Trait.cs
│   └── Economy/                   # Market, trading, shop
│       └── Market.cs
│
├── Combat/                        # (M3) — Sub-system
│   ├── Encounter/                 # CombatEncounter, CombatGrid
│   ├── Units/                     # CombatUnit + Movement/Combat/Health
│   ├── Skills/                    # Skill, SkillExecutor, Hitbox
│   │   └── HitShapes/            # IHitShape, Rectangle, Line, Cone, Cross
│   ├── StatusEffects/             # StatusEffect, ElementalReaction
│   ├── Terrain/
│   └── Damage/                    # DamageCalculator (formula từ JSON)
│
├── Persistence/                   # (M6) — Cross-run data
│   ├── RunSave.cs
│   └── ProfileSave.cs             # Achievements, legacy, knowledge flags
│
├── Meta/                          # (M6) — Cross-run logic
│   ├── Achievements/
│   ├── Legacy/                    # Avatar cũ → NPC
│   ├── CharacterCreation/         # Bonus, hack items, knowledge flags
│   └── MetaReincarnation.cs
│
├── Modding/                       # (M7)
│   ├── ModLoader.cs               # BepInEx integration
│   ├── HarmonyPatchTargets.cs
│   └── ModApi.cs                  # Public API cho modder
│
└── UI/                            # Parallel với mỗi milestone
    ├── Core/                      # UIManager, panel base
    ├── Sim/                       # Disciple list, building info
    ├── World/                     # World map UI
    ├── Combat/                    # Combat HUD
    └── Meta/                      # Main menu, char creation

Assets/Data/                       # JSON files — source of truth
├── units/
├── skills/
├── buildings/
├── items/
├── races/
├── traits/
├── recipes/
├── factions/
├── storytellers/
└── formulas/                      # damage formula, breakthrough cost...
```

---

## 17. Nguyên tắc kiến trúc

### 17.1. Data-driven (JSON là source of truth)
Mọi thông số gameplay (unit stats, skill damage, building output, recipe, trait) đều load từ JSON. Edit qua Odin Editor window (POCO class + OdinEditorWindow). Runtime load qua Addressables. Không dùng ScriptableObject.

**Workflow**: Odin Editor → edit POCO → save JSON → runtime load JSON → DataRegistry lookup by string ID.

Modder override: drop JSON file vào mod folder → ModLoader merge/override theo ID.

### 17.2. ID là string
Mọi entity (unit, building, trait, recipe, faction) có string ID. Không dùng int — dễ mod, dễ cross-reference trong JSON.

### 17.3. Tick-based sim
`GameClock` phát tick (Fast 10/s, Slow 1/s, Day). Mọi sim subscribe, không Update() rải rác.

### 17.4. EventBus xuyên hệ thống
Cross-system communication qua event, không direct coupling. Xem section 18.

### 17.5. Registry pattern
Không `FindObjectsOfType` trong hot path. Disciple, CombatUnit, Building tự đăng ký vào registry khi spawn, remove khi destroy.

### 17.6. Service Locator
Cho true singleton (EventBus, GameClock). Test-friendly (clear + register mock), swap-friendly (mod replace service).

### 17.7. Namespace
`XianXian.Core`, `XianXian.Stats`, `XianXian.Combat`, etc. — để Harmony patch dễ target.

### 17.8. Stat dùng enum
`Dictionary<StatType, float>` thay vì `Dictionary<string, float>`. Không magic string.

---

## 18. EventBus system

**Đã implement đầy đủ ở M1.**

### 18.1. Design core
- **GameEvent class** (không phải Action delegate) — có Timestamp, FrameNumber, IsCancelled, Cancel(reason)
- **Bus là instance**, không static — lấy qua ServiceLocator
- **3 scoped buses**:
  - `GlobalEventBus` — always-on (DiscipleDied, RealmBreakthrough, DayPassed)
  - `CombatEventBus` — tạo/hủy theo encounter (CombatUnitDied, SkillCast, DamageDealt)
  - `LocalMapEventBus` — tạo/hủy theo map (BuildingPlaced, ZoneCreated)

### 18.2. Priority + Cancellable
Handler có priority (int), cao chạy trước. Event có thể Cancel() giữa chừng.

**Quy ước priority**:
| Priority | Dùng cho |
|---------:|----------|
| +1000 | Pháp bảo intercept, cancel sớm |
| +500 | Buff/debuff protect (invincibility frame) |
| +100 | Core game logic (sect reputation, karma) |
| 0 | Default |
| -100 | UI update, audio cue |
| -1000 | Logger, analytics (chạy sau cùng) |

### 18.3. Auto-unsubscribe
`EventListenerBehaviour` base class cho MonoBehaviour:
- `ListenGlobal<T>()`, `ListenCombat<T>()`, `ListenLocalMap<T>()`
- Auto-unsubscribe khi OnDestroy — không memory leak

### 18.4. Logging & debug
- `EventLogger` interceptor — ring buffer 500 entry, filter theo type, console toggle
- Editor window: menu `XianXian → Event Debugger` — view realtime event log

### 18.5. Mod hook
- `IEventBusInterceptor` interface — modder register interceptor vào bus
- `OnBeforePublish()` + `OnAfterPublish()` — can thiệp event mà không cần Harmony patch

### 18.6. Bridge giữa bus
Combat unit chết → CombatEventBus fire `CombatUnitDiedEvent` → bridge service resolve disciple ID → GlobalEventBus fire `DiscipleDiedEvent`.

---

## 19. Data pipeline (JSON + Odin)

### 19.1. Workflow

```
[Dev] Mở Odin Editor Window
  → Load JSON từ disk → POCO object
  → Odin Inspector render POCO (edit như SO)
  → Save → JSON file cập nhật
  → Build → Addressables bundle JSON
  → Runtime → load JSON → POCO → DataRegistry<T>

[Modder] Drop JSON file vào mod folder
  → ModLoader merge/override theo ID
  → DataRegistry reflect changes
```

### 19.2. Asset reference trong JSON
JSON không serialize được Sprite/AnimatorController. Giải pháp:
- JSON lưu `"spriteAddress": "Sprites/disciple_basic"`
- Runtime load qua Addressables
- **Tất cả art asset phải đi qua Addressables**, không reference trực tiếp

### 19.3. Validation
- Editor-time: Odin window có validate button (check ID unique, reference valid)
- Runtime: validator riêng cho mod JSON (format check, missing field check)

---

## 20. Team vs Faction (architecture note)

- **Faction** = tổ chức chính trị tồn tại lâu dài ở world map. Đệ tử thuộc về Faction. Dùng cho meta logic (reputation, karma, war declaration).
- **Team** = phe trong một combat encounter cụ thể (gán lúc setup trận). Damage check dùng Team. Hậu quả meta (reputation drop khi giết đệ tử faction khác) dùng Faction.
- Một CombatUnit có **cả hai field**: `Faction faction` (di sản từ Disciple) + `Team team` (gán bởi encounter).
- AttackHitbox check `attackerTeam` — đúng. Faction lookup ở layer trên khi xử lý death event.


---
---

# PHẦN III — MILESTONE ROADMAP

---

## 21. Milestone overview

Mỗi milestone code compile được và prototype hiện tại vẫn chạy. Không big-bang — incremental delivery.

```
M1 ■■■■■■■■■■ DONE     Foundation (EventBus, Clock, Bootstrap)
M2 □□□□□□□□□□ NEXT     Data & Stats (JSON loader, DataRegistry, Odin Editor)
M3 □□□□□□□□□□           Combat refactor (port code cũ, IHitShape, DamageCalc)
M4 □□□□□□□□□□           Sim skeleton (Sect, Disciple, Building, LocalMap)
M5 □□□□□□□□□□           Cultivation (Realm, Tribulation, Karma, Reincarnation)
M6 □□□□□□□□□□           Meta layer (Achievement, Legacy, Save, CharCreation)
M7 □□□□□□□□□□           Modding hooks (BepInEx, ModLoader, ModApi)
```

---

## 22. M1 — Foundation (DONE ✅)

**Deliverables**:
- [x] Folder structure đầy đủ (placeholder cho M2-M7)
- [x] `Core/Enums/Faction.cs` + `Team.cs`
- [x] `Core/ServiceLocator.cs`
- [x] `Core/Time/GameClock.cs` — tick system (Fast 10/s, Slow 1/s, Day)
- [x] EventBus full:
  - [x] `GameEvent.cs` base class (Timestamp, Cancel, FrameNumber)
  - [x] `IEventBus.cs` interface + `IEventBusInterceptor`
  - [x] `EventBus.cs` implementation (priority sort, snapshot iteration, exception-safe)
  - [x] `ScopedBuses.cs` — GlobalEventBus, CombatEventBus, LocalMapEventBus
  - [x] `EventListenerBehaviour.cs` — auto-unsubscribe base MonoBehaviour
  - [x] `EventLogger.cs` — ring buffer 500 entry, filter, console toggle
  - [x] `Editor/EventDebuggerWindow.cs` — menu XianXian → Event Debugger
  - [x] `SampleEvents.cs` — 7 event mẫu (DiscipleDied, CombatUnitDied, etc.)
- [x] `Core/Bootstrap/GameBootstrap.cs` — wire mọi thứ, drop vào scene đầu
- [x] `Core/IsExternalInit.cs` — C# 9 shim (hotfix cho Unity runtime)

**Files**: 13 file C#, ~1045 dòng
**Output**: `XianXian_M1_Foundation.zip` (đã bao gồm hotfix)

**Cách test**: Scene Bootstrap → empty GO `_Bootstrap` → add GameBootstrap → Play → XianXian → Event Debugger

---

## 23. M2 — Data & Stats

**Mục tiêu**: hệ thống data-driven hoạt động end-to-end. Sau M2, mọi definition (unit, skill, building, trait) đều load từ JSON và edit được qua Odin Editor window.

**Deliverables**:
- [ ] `Data/Loaders/JsonDataLoader.cs` — đọc JSON từ disk/Addressables → POCO
- [ ] `Data/Loaders/DataRegistry.cs` — generic lookup: `DataRegistry<UnitDefinition>.Get("disciple_basic")`
- [ ] `Data/Definitions/` — POCO classes:
  - [ ] `UnitDefinition.cs` (port từ UnitData cũ + mở rộng)
  - [ ] `SkillDefinition.cs`
  - [ ] `EquipmentDefinition.cs`
  - [ ] `BuildingDefinition.cs`
  - [ ] `TraitDefinition.cs`
  - [ ] `RecipeDefinition.cs`
  - [ ] `FactionDefinition.cs`
- [ ] `Data/Editor/OdinDataEditorWindow.cs` — Odin EditorWindow đọc/ghi JSON trực tiếp
- [ ] `Stats/` — port + refactor từ code cũ:
  - [ ] `CharacterStats.cs` — Dictionary<StatType, float>, modifier stack
  - [ ] `StatModifier.cs` — additive + multiplicative
  - [ ] `Enums/StatType.cs`, `ModifierType.cs`, `CultivationRealm.cs`, `ElementAffinity.cs`
- [ ] `Equipment/` — port + refactor:
  - [ ] `EquipmentItem.cs`, `Inventory.cs`
  - [ ] `Enums/EquipmentSlot.cs`, `ItemRarity.cs`
- [ ] Sample JSON files:
  - [ ] `Assets/Data/units/` — 2-3 unit JSON
  - [ ] `Assets/Data/traits/` — 10-15 trait JSON (initial trait pool)
  - [ ] `Assets/Data/buildings/` — 12 building JSON (draft list ở section 6.1)
  - [ ] `Assets/Data/formulas/` — damage formula, breakthrough cost

**Khi xong M2**: mở Odin Editor window → thấy list definition → edit → save → JSON cập nhật. Code runtime load được.

---

## 24. M3 — Combat refactor

**Mục tiêu**: port BattleManager + UnitController + AttackHitbox cũ vào layout mới, tách thành components, thêm extensible hitshape.

**Deliverables**:
- [ ] `Combat/Encounter/CombatEncounter.cs` — đổi tên từ BattleManager, tạo/hủy CombatEventBus
- [ ] `Combat/Encounter/CombatGrid.cs` — grid ô vuông cho combat
- [ ] `Combat/Units/CombatUnit.cs` — orchestration only
- [ ] `Combat/Units/UnitMovement.cs` — tách pathfinding logic
- [ ] `Combat/Units/UnitCombatComponent.cs` — tách attack/skill logic
- [ ] `Combat/Units/UnitHealth.cs` — tách HP/die/takedamage
- [ ] `Combat/Skills/Skill.cs` + `SkillExecutor.cs`
- [ ] `Combat/Skills/Hitbox.cs` — đổi từ AttackHitbox
- [ ] `Combat/Skills/HitShapes/IHitShape.cs` — interface
- [ ] `Combat/Skills/HitShapes/RectangleShape.cs`, `LineShape.cs`, `ConeShape.cs`, `CrossShape.cs`
- [ ] `Combat/StatusEffects/StatusEffect.cs` + `StatusEffectController.cs`
- [ ] `Combat/StatusEffects/ElementalReaction.cs`
- [ ] `Combat/Damage/DamageCalculator.cs` — load formula từ JSON
- [ ] `Combat/Terrain/CombatTerrain.cs`
- [ ] Bridge: CombatUnitDied → DiscipleDiedEvent (via GlobalEventBus)

**Khi xong M3**: combat encounter chạy được trên grid, skill hit theo shape, damage tính từ JSON, event publish đúng bus.

---

## 25. M4 — Sim skeleton

**Mục tiêu**: khung quản lý tông môn hoạt động. Đệ tử có thể assign vào building, production tick, quest board.

**Deliverables**:
- [ ] `LocalMap/Grid/` — Tile, GridSystem (RimWorld-style 2D grid)
- [ ] `LocalMap/Buildings/Building.cs` — tier, production, requirement
- [ ] `LocalMap/Buildings/BuildingController.cs` — tick production từ đệ tử stats
- [ ] `LocalMap/Buildings/WorkAssignment.cs` — manual assign + standing order
- [ ] `LocalMap/Zoning/Zone.cs` + `ZoneType.cs`
- [ ] `LocalMap/Pathfinding/LocalPathfinder.cs` — A* wrapper
- [ ] `Sim/Sect/Sect.cs` — sect-level aggregate (resource, reputation, disciple count)
- [ ] `Sim/Sect/SectResources.cs` — track 5-8 core resources
- [ ] `Sim/Disciples/Disciple.cs` — CharacterStats + trait + assignment + desire + loyalty
- [ ] `Sim/Disciples/DiscipleRegistry.cs` — static registry (add on spawn, remove on die/leave)
- [ ] `Sim/Disciples/Recruitment.cs` — recruit UI logic
- [ ] `Sim/Disciples/Traits/Trait.cs` — load từ TraitDefinition JSON
- [ ] `Sim/Disciples/ExpectationSystem.cs` — track expectation per trait → trigger negative event
- [ ] `Sim/Economy/Market.cs` — fixed price buy/sell
- [ ] `Sim/Economy/Shop.cs` — player shop (passive income)
- [ ] Quest system skeleton:
  - [ ] `Sim/Quests/Quest.cs`
  - [ ] `Sim/Quests/QuestRegistry.cs`
  - [ ] `Sim/Quests/QuestBoard.cs` — per settlement

**Khi xong M4**: có local map grid, có building, có đệ tử assign vào building, production tick ra resource, có quest board, có market.

---

## 26. M5 — Cultivation systems

**Mục tiêu**: hệ thống tu tiên hoạt động. Cảnh giới, đột phá, thiên kiếp, tẩu hỏa, nhân quả, trùng sinh in-run.

**Deliverables**:
- [ ] `Cultivation/CultivationManager.cs` — quản lý cảnh giới per đệ tử
- [ ] `Cultivation/BreakthroughSystem.cs` — yêu cầu + risk + success formula
- [ ] `Cultivation/HeavenlyTribulation.cs` — trigger combat encounter khi đột phá cao
- [ ] `Cultivation/QiDeviation.cs` — tẩu hỏa nhập ma (desire + tâm ma tích tụ)
- [ ] `Cultivation/KarmaSystem.cs` — track hành động → ảnh hưởng event sau
- [ ] `Cultivation/Reincarnation.cs` — trùng sinh in-run (previousLifeId link)
- [ ] `Cultivation/LifespanSystem.cs` — aging + lifespan cap per cảnh giới
- [ ] Storyteller skeleton:
  - [ ] `Core/Storyteller/IStoryteller.cs` — interface
  - [ ] `Core/Storyteller/StorytellerBase.cs`
  - [ ] `Core/Storyteller/TianDaoStoryteller.cs` — Cassandra-style (rising tension)
  - [ ] `Core/Storyteller/NhanQuaStoryteller.cs` — action-reaction
  - [ ] `Core/Storyteller/HonDonStoryteller.cs` — Randy-style (random)
  - [ ] `Core/Storyteller/TuDinhStoryteller.cs` — sandbox (minimal events)
- [ ] Faction war resolution (abstract sim, không play-out từng trận)
- [ ] World map skeleton:
  - [ ] `World/WorldMap.cs` — grid ô vuông 300-500 tile
  - [ ] `World/WorldTile.cs` — linh khí score, resource type, owner faction
  - [ ] `World/Settlement.cs` — thành NPC (5-7 handcraft + 20-30 template)
  - [ ] `World/Region.cs`
  - [ ] `World/TravelSystem.cs` — time + resource + risk + random encounter
  - [ ] `World/CaravanOutfitting.cs` — pre-trip resource allocation

**Khi xong M5**: đệ tử tu luyện → đột phá → thiên kiếp → thăng cảnh giới. Storyteller push event. World map navigable. Faction war resolve.

---

## 27. M6 — Meta layer

**Mục tiêu**: save system + cross-run progression hoạt động. Achievement, legacy NPC, character creation.

**Deliverables**:
- [ ] `Persistence/RunSave.cs` — serialize toàn bộ run state
- [ ] `Persistence/ProfileSave.cs` — cross-run (achievements, legacy, knowledge flags)
- [ ] `Core/Save/SaveSystem.cs` — save/load/auto-save
- [ ] `Meta/Achievements/Achievement.cs` — condition + reward
- [ ] `Meta/Achievements/AchievementRegistry.cs` — track + unlock
- [ ] `Meta/Legacy/LegacyNpc.cs` — avatar cũ → NPC data
- [ ] `Meta/Legacy/LegacyRegistry.cs` — list legacy NPC available
- [ ] `Meta/CharacterCreation/CharacterCreation.cs` — bonus stat, hack items, knowledge flag selection
- [ ] `Meta/MetaReincarnation.cs` — trùng sinh meta-run (run end → new run start)
- [ ] Knowledge flag system:
  - [ ] `ProfileSave.UnlockedKnowledgeFlags` — HashSet<string>
  - [ ] JSON choice field `requiredKnowledge` — conditional choice display

**Khi xong M6**: game save/load được. Khi run kết thúc → achievement unlock → legacy NPC spawn ở run sau → character creation có bonus.

---

## 28. M7 — Modding hooks

**Mục tiêu**: modder có thể thêm content (JSON) và logic (C# via BepInEx) vào game.

**Deliverables**:
- [ ] `Modding/ModLoader.cs` — BepInEx integration, discover mod folder, load JSON override
- [ ] `Modding/ModApi.cs` — public API surface:
  - Register new event type
  - Register new storyteller
  - Register interceptor
  - Access DataRegistry (read-only)
  - Register new building/trait/recipe
- [ ] `Modding/HarmonyPatchTargets.cs` — đánh dấu method nào cam kết stable signature
- [ ] Asset loader cho mod:
  - Raw PNG → Sprite → register vào Addressables fake address
  - Raw JSON → POCO → merge vào DataRegistry
- [ ] Documentation:
  - [ ] `Modding/README_MODDERS.md` — hướng dẫn tạo mod cơ bản
  - [ ] Sample mod project (BepInEx plugin + JSON data)

**Khi xong M7**: modder có thể:
1. Drop JSON vào mod folder → add trait/building/storyteller
2. Drop PNG → add sprite
3. Write BepInEx plugin → hook vào EventBus, override game logic


---
---

# PHẦN IV — SETUP & OPS

---

## 29. Setup instructions

### 29.1. Tạo project Unity

1. Unity Hub → New Project
2. Template: **Universal 2D Core**
3. Unity version: **6 (6000.x)**

### 29.2. Project Settings
- Player → Active Input Handling = **Both**

### 29.3. Package Manager (built-in packages)
- Input System
- Addressables
- Cinemachine 3.x

### 29.4. Asset Store imports (theo thứ tự)
1. **Odin Inspector + Serializer** (paid)
2. **DOTween free** → sau import: Tools → Demigiant → DOTween Utility Panel → Setup DOTween → Apply
3. **A* Pathfinding Project Free** → sau import: AstarPath → Configure → Add Grid Graph (2D mode, node size = 1)

**KHÔNG cài**:
- DOTween Pro (chưa cần)
- Post Processing Stack v2 (URP có Volume sẵn)
- 2D Animation / PSD Importer (không dùng skeletal sprite)

### 29.5. Addressables init
- Window → Asset Management → Addressables → Groups → **Create Addressables Settings**
- Project Settings → Addressables → Build Remote Catalog = **false**
- Tạo group: `LocalContent` (default), `ModContent` (M7 sau)

### 29.6. Drop M1 code
- Giải nén `XianXian_M1_Foundation.zip` vào root project
- `Assets/Scripts/` xuất hiện
- Unity tự compile — Console phải sạch (không lỗi đỏ)

### 29.7. Test M1
- Tạo scene `Bootstrap.unity`
- Empty GO `_Bootstrap` → Add Component `GameBootstrap` (tự add GameClock)
- Play → menu **XianXian → Event Debugger** → window mở, không lỗi

---

## 30. Pitfall & lưu ý đã gặp

### 30.1. C# 9 `init` setter — lỗi CS0518
- **Lỗi**: `Predefined type 'System.Runtime.CompilerServices.IsExternalInit' is not defined`
- **Nguyên nhân**: Unity compile C# 9 nhưng runtime .NET Standard 2.1 thiếu type
- **Fix**: file shim `Core/IsExternalInit.cs` (đã bao gồm trong M1 zip)
- **Nếu gặp tương tự với `record`, `with`, `required`**: thêm shim tương ứng

### 30.2. Cinemachine 3 vs 2
- API khác hoàn toàn. Tutorial phải từ 2024+ mới valid.
- `CinemachineVirtualCamera` → `CinemachineCamera`

### 30.3. .meta files
- Mỗi file/folder có .meta kèm. Commit cùng nhau.
- Không commit .meta thiếu file hoặc ngược lại.

### 30.4. Asmdef (Assembly Definition)
- M1 chưa có .asmdef. Code compile vào Assembly-CSharp mặc định.
- Sẽ thêm khi cấu trúc ổn định (M2-M3):
  - `XianXian.Core.asmdef`
  - `XianXian.Combat.asmdef`
  - `XianXian.Sim.asmdef`
  - etc.

### 30.5. Folder `Editor/` trong Unity
- Folder tên `Editor` = Editor-only assembly, không build vào game.
- `EventDebuggerWindow.cs` nằm trong `Core/Events/Editor/` + có `#if UNITY_EDITOR` dự phòng.

### 30.6. Input System — chọn "Both" ban đầu
- Một số asset (Odin, DOTween debug) vẫn dùng old Input API
- Chọn "Both" → đổi "New only" khi mature

### 30.7. A* Pathfinding Free — giới hạn
- Không có local avoidance (đệ tử chồng lên nhau)
- Không multithread (path lớn block main thread)
- Upgrade Pro sau M4 khi 50+ đệ tử pathfind

---

*Tài liệu cập nhật lần cuối: sau M1 hoàn tất + gameplay design discussion (H/E/B/D/F).*
*Phiên bản tiếp theo: update sau khi M2 hoàn tất.*

---
---

# APPENDIX C — Bổ sung thiết kế (sau M1)

> Thông tin này bổ sung / điều chỉnh các section ở Phần I. Khi có conflict, Appendix C là source of truth mới nhất.

---

## C.1. Hệ thống Chỉ số & Phân cấp Nhân vật (bổ sung section 5)

### C.1.1. Độ hiếm & Tổng điểm bẩm sinh (Stat Pool)

Khi sinh ra / chiêu mộ, mỗi đệ tử được phân bổ ngẫu nhiên tổng điểm từ pool vào **6 chỉ số gốc** dựa trên độ hiếm:

| Độ hiếm | Tổng pool | Ghi chú |
|---|---|---|
| Thường nhân | 200 | Đa số đệ tử |
| Tài năng | 280 | Khá phổ biến |
| Kỳ tài | 350 | Hiếm |
| Thiên tài | 450 | Rất hiếm — sharp distribution |

Phân bổ random từ 0-100 cho mỗi chỉ số trong tổng pool trên.
→ Một Thiên tài 450 điểm vẫn có thể có Mị lực = 5 nếu toàn bộ pool dồn vào Lực đạo + Ngộ tính.

### C.1.2. 6 Chỉ số Gốc (Core Stats — gần như cố định)

**Đây là chỉ số cứng, bẩm sinh, khó thay đổi.** Chúng quyết định combat stats (derived) và hiệu suất làm việc. Không thể tăng trực tiếp bằng trang bị hay công pháp — chỉ có vài event đặc biệt hoặc secret technique mới thay đổi được.

| Chỉ số | Tên tiếng Anh | Tác động Combat | Tác động Công việc |
|---|---|---|---|
| **Tư chất** | Aptitude | Multiplier tổng lực chiến; quyết định growth rate khi lên cảnh giới | Không ảnh hưởng trực tiếp |
| **Lực đạo** | Force | Tăng ATK | Đốn củi ↑, Luyện khí ↑ |
| **Căn cốt** | Foundation | Tăng HP, DEF, khả năng sinh tồn | Khai khoáng ↑ |
| **Thân pháp** | Agility | Tăng SPD (hồi cooldown, né tránh, di chuyển), Crit Rate | Luyện đan ↑ (tỷ lệ + giảm thời gian) |
| **Ngộ tính** | Comprehension | Tăng tốc độ học skill, chỉ số nội tại | Hái thuốc ↑, Luyện đan ↑, Luyện khí ↑, Vẽ bùa ↑ |
| **Mị lực** | Charisma | Hỗ trợ ngoại giao, đàm phán | Vẽ bùa ↑ |

### C.1.3. Chỉ số Combat (Derived — có thể tăng)

Derived từ Core Stats + modifier từ **công pháp được học** + **trang bị**. Trang bị và công pháp **KHÔNG** ảnh hưởng tới hiệu suất làm việc (chỉ Core Stat mới ảnh hưởng việc).

| Chỉ số | Nguồn gốc |
|---|---|
| ATK | Lực đạo (base) + Công pháp + Trang bị |
| DEF | Căn cốt (base) + Công pháp + Trang bị |
| HP | Căn cốt (base) + Công pháp + Trang bị |
| SPD | Thân pháp (base) + Công pháp + Trang bị *(SPD không giảm Cast Time)* |
| Crit Rate | Thân pháp (base) + Công pháp + Trang bị |
| Crit DMG | Công pháp + Trang bị (base 150%) |
| Hit Accuracy | Công pháp + Trang bị |
| Evasion | Thân pháp (base) + Công pháp + Trang bị |
| Lifesteal | Công pháp + Trang bị |
| Armor Pen | Công pháp + Trang bị |

### C.1.4. Hiệu suất Làm việc (Work Output)

Work output **CHỈ** có thể tăng qua 2 cách:
1. **Tăng skill level** làm việc của đệ tử (học thêm, lên cấp kỹ năng)
2. **Nâng tier công trình** lên cao hơn

Trang bị và công pháp **không** ảnh hưởng work output.

### C.1.5. Linh căn (Elemental Affinity — bổ sung section 5.2)

**4 hệ cơ bản**: Phong, Thủy, Hỏa, Thổ — mở khóa học công pháp đồng hệ.

**4 biến dị** (rarer, stronger): Lôi, Băng, Kim, Mộc.

**Tiền tố "Thiên"** (cực hiếm): Thiên Hỏa, Thiên Lôi, v.v. — tăng mạnh sát thương/hiệu ứng nguyên tố.

**Cơ chế khắc chế**: dựa trên chỉ số Kháng nguyên tố (resistance stat), không phải hard counter.

### C.1.6. Vòng đời bổ sung

- **Permadeath**: Hết thọ nguyên mà không đột phá cảnh giới → chết vĩnh viễn (không thể revive).
- **Thú cưỡi / Linh sủng**: Entity riêng, cần thức ăn đặc biệt để tăng EXP — không phải đệ tử thường.

---

## C.2. Hệ thống Trang bị (bổ sung section 6)

### C.2.1. 9 Vị trí trang bị

| # | Slot | Ghi chú |
|---|---|---|
| 1 | Headwear | Mũ / Khăn |
| 2 | Armor | Áo giáp |
| 3 | Leggings | Quần giáp |
| 4 | Gloves | Găng tay |
| 5 | Boots | Giày |
| 6 | Jewelry 1 | Nhẫn |
| 7 | Jewelry 2 | Ngọc bội |
| 8 | **Weapon** | Bắt buộc để đánh thường + dùng skill công pháp. Quyết định tầm đánh: Kiếm = gần, Cung = xa, Quạt = phép |
| 9 | **Artifact** (Pháp bảo) | **Chỉ Key Unit** mới được dùng. Cung cấp stats + 1 Active Skill mạnh |

### C.2.2. Phẩm chất item (6 tier)

| Tier | Màu | Tên | Ghi chú |
|---|---|---|---|
| 1 | Trắng | Common | Phổ thông |
| 2 | Lục | Uncommon | |
| 3 | Xanh | Rare | |
| 4 | Tím | Epic | |
| 5 | Cam | Legendary | |
| 6 | Đỏ | Unique | Hiệu ứng độc nhất, chỉ số có thể rất lệch (build-defining) |

### C.2.3. Cấu trúc item

- **2-4 thuộc tính chính** (Main stats)
- **Tối đa 4 thuộc tính phụ** (Sub-stats) — Prefix/Suffix quyết định bởi tay nghề thợ Luyện khí
- **Set trang bị**: bonus khi mặc đủ bộ
- **Yêu cầu trang bị**: giới hạn theo Level, Cảnh giới, hoặc Linh căn

---

## C.3. Kỹ năng & Combat (bổ sung section 8)

### C.3.1. Skill Slots

Số lượng slot tăng dần theo Tu vi:

| Slot | Loại | Ghi chú |
|---|---|---|
| 1 | Đánh thường | Gây sát thương thuần trong lúc chờ cooldown |
| 2-3 | Công pháp chủ động (×2) | Skill tấn công / hỗ trợ học từ Tàng Kinh Các |
| 4 | Nội tại (Passive) | Luôn active, không cần cast |
| 5 | Pháp bảo skill | **Chỉ Key Unit** — 1 Active Skill từ Artifact slot |

### C.3.2. Học Skill

- Học thông qua **Tàng Kinh Các** (Library building)
- Player gán skill **thủ công** cho từng đệ tử — không auto-assign
- Giới hạn số lượng skill có thể học + tốc độ học phụ thuộc vào **Ngộ tính**

### C.3.3. Thi triển (Casting)

- Các chiêu mạnh có **Cast time** (Niệm chú) — đứng yên trong lúc cast
- Có thể bị **ngắt** (interrupt / Stun) — implement qua EventBus + Priority system
- **SPD không làm giảm Cast time** (SPD chỉ ảnh hưởng cooldown recovery + evasion + di chuyển)

---

## C.4. Hệ thống Traits — Bổ sung (bổ sung section 5.2)

### C.4.1. Nguồn gốc Trait

1. **Bẩm sinh**: đổ ngẫu nhiên khi sinh ra / chiêu mộ
2. **Event Encounter**: nhận thêm khi đi map, gặp sự kiện đặc biệt
3. **Hidden Achievement** (mới): đạt điều kiện ẩn trong game → tự ngộ ra trait mạnh

### C.4.2. Hidden Achievement Trait — Ví dụ

| Điều kiện | Trait nhận được | Effect |
|---|---|---|
| Học đủ 3 kiếm pháp cấp siêu việt | **Kiếm Thánh** | Buff mạnh sát thương bạo kích kiếm |
| (expand khi thiết kế thêm) | | |

### C.4.3. Tác dụng Trait (chi tiết hơn section 5.2)

- Tăng kinh nghiệm làm việc (work EXP gain rate)
- Cộng thẳng sát thương vào combat
- Thêm hiệu ứng chiến đấu (on-hit, on-crit, etc.)
- Unlock kỹ năng đặc biệt (skill mới không học được từ Tàng Kinh Các)

---

## C.5. Architecture notes cho M2+ (implications)

### Stat architecture

```
CoreStats (6 — gần cố định)
  └─ drives ─► CombatStats (derived — tăng được qua công pháp + trang bị)
  └─ drives ─► WorkEfficiency (derived — chỉ tăng qua skill level + building tier)

StatType enum chia 3 nhóm:
  - Core (Aptitude, Force, Foundation, Agility, Comprehension, Charisma)
  - Combat (ATK, DEF, HP, SPD, CritRate, CritDmg, HitAcc, Evasion, Lifesteal, ArmorPen)
  - Work (WorkSkillLevel — per activity, không phải per stat)
```

### Equipment architecture

```
EquipmentSlot (9):
  Headwear, Armor, Leggings, Gloves, Boots,
  Jewelry1, Jewelry2, Weapon, Artifact

ItemRarity (6):
  Common, Uncommon, Rare, Epic, Legendary, Unique

EquipmentDefinition:
  - id, slot, rarity
  - mainStats (2-4 entries)
  - subStats (0-4 entries, determined by crafting quality)
  - setId (nullable)
  - requireLevel, requireRealm, requireElement
  - artifactSkillId (nullable — only for Artifact slot)
```

### Key Unit concept

Key Unit = avatar player hoặc đệ tử được chỉ định làm "key" cho một trận.
- Được trang bị Artifact slot
- Được dùng Pháp bảo skill (slot 5)
- Chỉ 1 Key Unit per encounter (có thể expand sau)

---

*Appendix C thêm vào: sau thảo luận bổ sung về Stats/Equipment/Combat system.*

