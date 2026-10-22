---
name: project-city-real-buildings
description: "City gen 2026-07-31 — công trình thành 'nhà thật' (tường/phòng/cửa từng ô) + template quota bị bỏ quên đã nạp lại; bảng quy mô map theo số nhà"
metadata: 
  node_type: memory
  type: project
  originSessionId: 12b41f14-d079-44a7-8aef-750c387c40b1
  modified: 2026-08-01T11:15:01.790Z
---

**CITY "NHÀ THẬT" (2026-07-31)** — trước đó mọi công trình thành phố chỉ là 1 hình chữ nhật `Occupant`
ĐẶC (không tường/cửa/ruột, `Entrance` = quy ước "NPC tới rồi biến mất"). Giờ mỗi căn có **tường bao +
phòng + cửa ở cấp độ TỪNG Ô**, đi vào trong được như nhà Wild.

**Code mới:** `World/City/BuildingInterior.cs` (Walls/Floors/Doors/FrontDoor/Rooms, immutable) +
`World/City/Generation/BuildingFloorPlan.cs` (BSP chẻ phòng). `PlacedBuilding.Interior` (null = khối đặc)
gắn ở **PASS RIÊNG cuối `CityGenerator.Generate` (Step 6)** — tạo `PlacedBuilding` MỚI qua `WithInterior`
nên vẫn immutable. TileType MỚI append-only: `Floor = 22`, `Wall = 23` (+2 dòng trong `terrain_tiles.json`
CẢ 2 thư mục, + `GridSystem.IsPassableType` phải thêm `Wall => false` — hàm đó hardcode, KHÔNG đọc JSON).
Test: menu `Run M11 Interior Tests` (11 case). `RoomDetector` (Wild M15) tái dùng được nguyên vì chỉ cần
`GridSystem`.

**BẪY LỚN NHẤT — "cửa nhà không dính đường" (user phát hiện bằng MẮT 2026-07-31, test cũ vẫn xanh):**
1-6 căn mỗi bản đồ có cửa mở ra bãi cỏ. Đuổi theo 2 lớp mới tới gốc:
- Lớp 1 (chữa ngọn, chỉ giảm được ~1/3): thứ tự ưu tiên chọn cửa sai — "đúng mặt tiền" (+100) đè lên
  "mở ra đường" (×10). **Điểm ĐƯỜNG phải đứng TRƯỚC mặt tiền**: nhà dãy chọn hướng theo cạnh block chứ
  không theo "cạnh nào có phố", nên cửa phải tự đổi cạnh lấy. Đổi cạnh thì **`PlacedBuilding.Entrance` +
  `FacingDirection` phải kéo theo** không thì NPC pathfind tới cạnh cũ rồi đứng trước tường kín.
- Lớp 2 (GỐC THẬT): **`BlockBuildingPlacer.FaceTouchesRoad` tính cả 2 ô GÓC là mặt tiền.** Góc là chỗ 2
  bức tường gặp nhau — đặt cửa ở đó thì ô ngay phía trong lại là tường bên, vĩnh viễn không mở được. Nên
  có những căn CHỈ chạm đường đúng 1 ô góc vẫn được đặt, và không cửa nào cứu nổi. Sửa: bỏ 2 ô góc khỏi
  cả `FaceTouchesRoad` LẪN `FindEntrance` → nhà không có mặt tiền thật thì KHÔNG ĐẶT (chỗ đó thành sân).
  Sau khi sửa: **0/3.500 căn sai** trên 10 cấu hình; tổng số nhà gần như không đổi (packer tự lấp cách khác).
- **`M11CityTest` Test 5 "Frontage rule" chính là thứ bỏ lọt** — nó cũng đếm ô góc. Đã siết cả 2 test:
  frontage phải là ô viền KHÔNG GÓC, và interior test đổi từ "cửa mở ra ô ĐI ĐƯỢC" (quá lỏng — bãi cỏ
  cũng đi được) sang **"mở thẳng ra ĐƯỜNG" 100%** + "Entrance kề đúng cửa chính".
- **Bài học chung: test nới lỏng một bậc ("đi được" thay vì "là đường", "kề" thay vì "kề ở chỗ dùng được")
  là chỗ bug trốn.** Cả 2 lần đều là user nhìn ảnh mới ra.

**3 BẪY HÌNH HỌC khác (đều là lỗi IM LẶNG, mắt không thấy trên map 160×160):**
1. **Đặt cửa thông phòng PHẢI làm SAU khi chẻ xong hết**, không phải trong lúc chẻ — cửa trên vách CHA dễ
   mở thẳng vào vách CON (vách con chỉ tồn tại sau lần chẻ sâu hơn) → phòng kín tịt.
2. **Cửa chính phải xét CẢ 4 CẠNH, không chỉ mặt tiền** — có căn cả mặt tiền chỉ chạm đường đúng ô GÓC,
   mà góc không đặt cửa được (ô trong của nó là tường viền). Gặp thật 2/173 căn. `CityGenerator` truyền
   `outsideScore` (2=đường, 1=đất trống, 0=chặn) vào; cửa KHÔNG BAO GIỜ được mở ra ô điểm 0.
3. Vẫn giữ **BFS vá liên thông** làm lưới an toàn cuối cùng (đục thêm cửa trên vách ngăn vùng chưa tới).

**`roomMaxSize` = 6 chứ không phải 8:** với `minRoomSize` 3, một trục chỉ chẻ được khi ruột ≥ 7 (footprint
≥ 9). Để 8 thì castle 10×10 rơi vào vùng "chẻ theo may rủi" và đo thật ra **đúng 1.0 phòng/căn trên toàn
thành**. Nhà dân 6×5 (ruột 4×3) là nhà MỘT GIAN — cố ý, đúng nhà nhỏ ngoài đời.

**BUG LỚN phát hiện nhân tiện — thành phố KHÔNG HỀ có chợ/thư viện/doanh trại:** `SettlementTemplate` chỉ
đến từ 1 TextAsset kéo tay trên `CitySceneBootstrap`, mà ô đó để TRỐNG → `PlaceCompounds` không có gì để
đặt → chỉ toàn nhà dân từ filler pool. Không log gì cả, chỉ lộ ra khi ĐẾM công trình theo loại (6 loại →
**34/40 loại** sau khi sửa). Sửa: `CityGenConfig.settlementTemplateId` (mặc định `"template_town"`),
`CityGenerator` tự nạp `Data/city_templates/{id}.json` khi caller không truyền. File mới `template_town.json`
(map 160, triết lý: template CHỈ bảo đảm công trình CÔNG CỘNG, nhà ở nền để filler lấp);
`template_town_400.json` cũ nay là template của preset `large_city`; `village` để `""`.
**Test dùng def rút gọn PHẢI set `config.settlementTemplateId = ""`** không thì cả rừng warning
"definition không tìm thấy".

**ĐỢT 2 — HÌNH DẠNG + PHÓNG TO (2026-08-01, user chốt "nhà ×2 tuỳ tier, công trình lớn ×3, thêm nhà
chữ L / 2 căn, đừng vuông vức thiếu sức sống"):**
- **`BuildingShapes.cs` (mới)** — `rect`/`L`/`U`/`courtyard` (tứ hợp viện)/`twin`, khai ở
  `CityBuildingDefinition.shape`. Khung bao nhỏ → tự rơi về `rect`. Đo: **221/448 căn không phải
  hình chữ nhật**. Phần bị khoét KHÔNG vào footprint nhưng VẪN đánh dấu occupied (là SÂN của chính
  căn đó, không cho hàng xóm lấn).
- **3 luật bất biến của hình** (có test riêng): ① liên thông 4-hướng (rời khối ⇒ không nối được
  phòng, vá liên thông bó tay) ② **cạnh quay ra phố luôn ĐẶC** ③ không đủ chỗ → `rect`.
  *Test bắt lỗi thật ngay lần chạy đầu:* `twin` tách theo cạnh DÀI ⇒ khi cạnh dài trùng hướng phố
  thì khe hở cắt ngang mặt tiền. **Trục tách phải VUÔNG GÓC mặt phố** (nhà chính bám phố, nhà phụ
  lùi sau).
- **`BuildingFloorPlan` bỏ hẳn giả định chữ nhật:** tường bao = **đường viền của TẬP Ô** (ô có hàng
  xóm 4-hướng ngoài footprint) ⇒ tứ hợp viện tự bao cả mép sân trong, không cần code riêng. BSP vẫn
  chạy trên khung bao nhưng **đếm ô ruột THẬT trước mỗi nhát chẻ** (đường thẳng cắt hình lõm đẻ mảnh
  vụn). Cửa chính quét **mọi ô tường** thay vì quét theo cạnh — **ô góc tự loại** vì không có hàng
  xóm ruột phía trong (bài toán góc hôm trước tự tan). Thêm `isOwnYard` → **cổng ra sân riêng**,
  không thì sân chữ L / tứ hợp viện là mảnh đất không lối vào.
- **`BuildingInterior.Rooms`: `List<RectInt>` → danh sách Ô** (thành phần liên thông của `Floors`) —
  nhà chữ L thì phòng cũng không phải chữ nhật, `RectInt` sẽ nói dối về hình dạng.
- **`roomMaxSize` per-definition** (0 = theo config). BẮT BUỘC với công trình lớn: thuật toán chẻ tới
  khi mọi phòng ≤ ngưỡng ⇒ diện tích ×9 thì số phòng ×9. Đo thật: thư viện 36×30 với ngưỡng nhà dân
  ra **42 phòng bé giống hệt nhau = tổ ong**, thành chủ 30×30 ra 34 phòng. **Kích thước PHÒNG ⟂ kích
  thước CÔNG TRÌNH, không dùng chung một con số.** Khai riêng xong: thành chủ 11 phòng, chợ lớn 6.
- **Cỡ mới:** nhà dân phân hoá theo tier 10×8 (nghèo) → 14×12 (giàu) · cửa hàng 12×10 · lò rèn 20×14 ·
  nhà gia tộc 18×16 tứ hợp viện · chợ/doanh trại 34×22 · thư viện lớn 34×28 · thành chủ 30×30.
  Map: default **400** (~460 công trình, ~1320 phòng) · `large_city` **512** · `village` **200**.
- **3 ràng buộc dây chuyền dễ quên (đều hỏng IM LẶNG):** ① `blockMinSize` ≥ chiều NHỎ công trình to
  nhất (28) — nhỏ hơn thì compound không đặt được, chỉ 1 dòng warning (đo: ×3 ở map 256 chỉ đặt
  được 5/9) ② `compoundMinDim` > chiều nhỏ NHÀ DÂN (14) — nhỏ hơn thì mỗi nhà dân ăn trọn 1 lô
  ③ **`SettlementTemplate.coreRadius` GHI ĐÈ `CityGenConfig.coreRadius`** — đổi map mà quên sửa
  template thì Core zone teo theo số cũ. `template_town_400.json` đã đổi tên →
  `template_city_large.json` (preset large_city).
- **BẪY POWERSHELL (tự dính, làm hỏng 80 file JSON):** PowerShell KHÔNG phân biệt hoa/thường ⇒ biến
  `$t` (nội dung file) GHI ĐÈ bảng dữ liệu `$T`, rồi lỗi non-terminating không dừng script nên nó
  ghi tiếp giá trị RỖNG vào cả 80 file. Luật: trong script sửa data hàng loạt, **đặt tên biến khác
  nhau rõ rệt** và **luôn có bước verify cuối** (parse lại toàn bộ + check field + check encoding).

**ĐỢT 3 — LẤP CHỖ TRỐNG GIỮA CÁC CÔNG TRÌNH (2026-08-01, GĐ1 "quy hoạch"):**
- **Đo trước khi sửa mới ra đúng bệnh** (user: *"lấy kết quả thực tế luôn tốt hơn phỏng đoán"* — đúng,
  2/3 giả thuyết ban đầu của mình SAI). Công cụ quyết định là 1 dòng log trong `CityGenerator`:
  **tổng diện tích lô ⟂ diện tích đã xây, kèm histogram theo CHIỀU SÂU lô**. Không có nó thì mọi
  phán đoán về chỗ trống đều là đoán mò. **Giữ log này lại.**
- Kết quả đo: 87 lô, **chỉ 61% đất lô được xây**. Histogram lộ ra bệnh thật: lô sâu 0-9 (24 lô) lấp
  **0-25%** — quá nông, không nhà nào vừa (nhà nông nhất sâu 8).
- **Giả thuyết SAI #1: "AlleySplit chỉ chẻ 1 lần".** Đọc code thì nó ĐÃ đệ quy sẵn (Queue).
- **Sửa 1 — cắt LÁT từ một đầu thay vì CHẺ ĐÔI** (`AlleySplit`): chẻ đôi lô sâu 30 ra hai lô sâu 14-15,
  mà hai dãy lưng-kề-lưng cần đúng 2×10=20 → lô 14 chỉ chứa nổi MỘT dãy. Cắt lát đúng `alleyDepthMax`
  thì phần lớn lát rơi vào bội số chiều sâu nhà, chỉ còn ĐÚNG MỘT phần dư.
- **Sửa 2 — `minLotDepth` (config mới, =8 = chiều sâu nhà nông nhất)**: chặn nhát cắt đẻ ra lô nông
  vô dụng. Nhóm lô sâu 5-9 nhảy từ **25% → 83%** lấp đầy.
- **Sửa 3 — lô CÓ COMPOUND cũng được chẻ hẻm** (tham số `blocked` = ô đã có công trình, hẻm tự né;
  không cắt sạch được thì giữ lô nguyên). Trước đó nhóm này bị `if (usedBlocks.Contains(i))` bỏ qua
  hoàn toàn → 1 compound 20×14 giữ nguyên cả lô 60×60, **chiếm 45% tổng đất trống**.
- **Sửa 4 — hẻm thử CẢ TRỤC CÒN LẠI khi trục chính bị compound chắn.** Không có bước này thì lô 50×50
  chứa compound vẫn kẹt ở ~40% lấp đầy; sau khi thêm, nhóm lô sâu 45+ biến mất hẳn.
- **Sửa 5 — `DrawRowDef` lọc theo CHIỀU SÂU CÒN TRỐNG** thay vì rút mù rồi hụt. `FreeDepthAt` đếm ô
  trống từ mép lô ăn vào; chỉ rút def có `footprintHeight ≤` số đó.
- **Giả thuyết SAI #2 (đã thử, đo, rồi GỠ): "chừa chỗ cho dãy đối diện"** — giới hạn chiều sâu dãy
  ĐẦU = `lotDepth - minLotDepth` để dãy lưng còn chỗ. Nghe rất hợp lý. A/B 3 seed: **543 vs 539 nhà =
  nằm trong nhiễu**, nhóm lô sâu 15-19 vẫn 64-66% cả khi bật lẫn tắt. Đã gỡ + để lại comment cảnh báo
  trong `PackSide`. **Nhóm 15-19 vẫn chưa rõ nguyên nhân** — ai làm tiếp thì đo trước.
- **KẾT QUẢ (map 400, cùng thước đo, seed 2026):** công trình **463 → 546** (+18%) · đất trống
  **26,4% → 19,2%** (−27%) · lấp đầy đất lô **61% → 68%** · 0 cửa sai · test 16/16 + 16/16.
  Đuôi phân bố mảnh trống co lại rõ (tổng 10 mảnh to nhất 9773 → 7227 ô) nhưng **vẫn còn 1 mảnh
  ~1463 ô và 84% đất trống vẫn nằm trong mảnh ≥100 ô** — đây chính là phần dành cho GĐ2 (biến mảnh
  trống thành SÂN CÓ CHỦ ĐÍCH: vườn rau/bãi sạp/sân giếng theo zone), CHƯA làm.

**ĐỢT 3b — GĐ2 "SÂN CÓ CHỦ ĐÍCH" + ĐẤT CHỪA CHO NGƯỜI CHƠI (2026-08-01):**
- **Ràng buộc user chốt: người chơi phải có đất tự dựng nhà, KHÔNG buộc mua lại nhà NPC.** Vì thế
  `CityOpenLot.IsVacant` được chọn **TRƯỚC** khi gán vườn/bãi — nhặt phần thừa thì mọi mảnh đẹp đã
  bị gán loại mất. Bảo đảm bằng CẢ tỉ lệ (`vacantLotRatio` 0.30) LẪN sàn cứng (`minVacantLots` 6),
  và rải ĐỀU trên danh sách đã sắp (dồn hết vào 1 góc thành phố = coi như không có đất).
- **"Có lô trống" CHƯA ĐỦ — lô phải DÙNG ĐƯỢC:** (a) khung bao chứa lọt căn nhà nhỏ nhất, (b) có
  mặt tiền đường ở ô KHÔNG PHẢI GÓC (cùng luật `FaceTouchesRoad`). **Bug thật đã dính:** thước đo
  "nhà nhỏ nhất" quét mọi def `near_road` không-compound và vớ phải **THÁP CHUÔNG 8×8** (nhỏ hơn
  nhà dân 10×8) → sinh ra lô "chừa cho player" rộng 8×8 mà không nhà nào dựng lên được. Sửa: chỉ
  tính role Residence/Shop. **Đất chừa mà không dùng được thì bằng không.**
- Code mới: `CityLotDefinition` (JSON `Data/city_lots/`, 8 loại: vườn rau/sân giếng/sân phơi/bãi
  sạp/bãi xe ngựa/bãi nguyên liệu/vườn cảnh/sân diễn võ — lọc theo zone + minArea/maxArea + trọng
  số) · `CityOpenLot` (frozen) · `CityLotPlanner` (gom mảnh liền khối → chọn lô trống → gán loại) ·
  `CityLayout.OpenLots` + `SetOpenLots` (tách khỏi ctor để 2 ctor cũ không vỡ) · stamp `TileType`
  theo def (lô trống = `Dirt`, **buildable** theo `terrain_tiles.json`) · renderer sơn trên layer
  GROUND (lô là mặt đất, không phải công trình).
- Config mới: `planOpenLots` · `minLotArea` (12 — khe 2-3 ô là KHE, gán loại cho nó chỉ tạo rác dữ
  liệu) · `vacantLotRatio` · `minVacantLots`.
- **Kết quả (map 400):** 116-132 lô/thành, trong đó **16-23 lô chừa cho player** (~5% diện tích nội
  thành), 100% lô trống tự kiểm được là dựng nhà lên được. Test **M11 City 20/20** (thêm Test 12-14:
  lô không đè đường/nhà + rời nhau + liền khối · lô trống dùng được + đủ sàn · lô có loại đúng
  diện tích) + Interior 16/16. Đã verify Play Mode thật.
- **Test def của `M11CityTest` phải bám sát JSON thật** — Test 13 lấy "nhà ở nhỏ nhất" từ chính bộ
  def đó làm thước đo, để cỡ cũ (6×5) thì test vẫn xanh nhưng không còn phản ánh game thật.

**HỢP NHẤT CITY ↔ WILD — B1 THĂM DÒ XONG (2026-08-01). User chốt hướng: MỘT hệ duy nhất, Wild chỉ
là nơi prototype trước cho có nội dung.**
- **KHÔNG copy code Wild sang City** (copy = 2 bản phân kỳ, sửa bug 2 nơi). Cách đúng: **City sinh
  ra `LocalMapData`** rồi dùng nguyên runtime Wild. File mới: `World/City/Generation/
  CityLocalMapAdapter.cs` — **chỉ ~25 dòng thật**, vì `LocalMapData` ctor = `(GridSystem, seed,
  biomeId)` và mọi list còn lại mặc định rỗng.
- **NỀN THÀNH PHỐ VẼ ĐƯỢC NGAY, KHÔNG SỬA DÒNG NÀO trong `WildTilemapRenderer`** — nó tra màu theo
  `terrain_tiles.json.debugColor`, mà city đã stamp `Wall`/`Floor`/`Road`/`Dirt`/`Farmland`/`Stone`
  và cả 6 loại đều có sẵn trong file đó. Đây là lợi tức bất ngờ của quyết định "stamp nội thất
  xuống TileType" hôm 2026-07-31.
- **ĐO HIỆU NĂNG — KẾT LUẬN NGƯỢC VỚI LO NGẠI BAN ĐẦU:** map 400×400 (160k ô).
  Wild renderer: gen 245ms · adapt 1ms · **render 629ms (1 lần)** · chạy ổn định **12 FPS** (86ms/frame);
  bật thêm overlay vai trò → 7 FPS. **`CityTilemapRenderer` HIỆN TẠI, cùng map, cùng 546 công trình:
  cũng 7 FPS (153ms).** ⇒ **Lag KHÔNG do việc port — nó đã có sẵn từ trước, và bản Wild còn NHANH
  HƠN.** Nguyên nhân thật: cả 2 renderer đều **ép `TilemapRenderer.Mode.Individual` trong code** (để
  `SetColor` từng ô) ⇒ 160k sprite rời, mất hết batching.
- **Đường sửa hiệu năng (chưa làm, chưa đo):** `WildTilemapRenderer` ĐÃ có sẵn nhánh sprite
  (`_groundTiles`: TileType → TileBase, doc ghi "SPRITE-READY"). Dùng tile asset thật thay
  white-tile+SetColor thì bỏ được Individual mode. **Giả thuyết, phải đo trước khi tin.** Nhưng nó
  là đường duy nhất có sẵn ⇒ port sang Wild renderer chính là thứ MỞ đường sửa.
- **Mất màu theo vai trò là thật** (renderer Wild tô theo TileType → cả thành phố còn 2 màu
  tường/sàn). Giải pháp user chốt: **nút filter debug**. Đã làm trong probe — vẽ lớp phủ trên
  **tilemap RIÊNG của probe**, KHÔNG thêm API vào `WildTilemapRenderer` (mọi hàm highlight của nó
  đều `ClearHighlight()` trước = ngữ nghĩa "một mục tiêu"; nhét chế độ nhiều-màu vào là bẻ hợp đồng
  của nó chỉ vì một công cụ debug).
- Scene thử RIÊNG `Assets/Scenes/CityWildProbe.unity` + `World/City/CityWildRenderProbe.cs` (dựng
  bằng MCP, không kéo tay). Không đụng `CityScene.unity`. Xoá được sau khi B2-B4 xong.
- **CÒN LẠI (B2-B4, chưa làm):** ① tách nguồn dữ liệu khỏi `WildSceneBootstrap` — **14 chỗ**
  SerializeField bám cứng nó (`PawnAiBrain`, `WildSceneInteraction`, `WildHUDController`, gather/
  beast/production/combat…) ② `CityBuildingProvider`/`CityLotProvider` cắm vào `ITileInteractableProvider`
  (10 provider đã có sẵn — đúng điểm cắm đã thiết kế, không phải mổ hệ) ③ port 4 thứ City-only lên
  panel Wild: minimap · zone overlay · building info panel · **hệ job kiếm tiền** (`CityJobBootstrap`).

**B2 XONG (2026-08-01) — `LocalMapHost`, City và Wild giờ CÙNG một hợp đồng:**
- `World/LocalMapHost.cs` — **LỚP CHA MonoBehaviour, CỐ Ý KHÔNG PHẢI INTERFACE. Đừng "dọn" thành
  interface:** Unity không serialize tham chiếu interface. Với lớp cha, chỉ cần NỚI kiểu field của
  consumer từ `WildSceneBootstrap` → `LocalMapHost` và **mọi wiring đã kéo tay trong `WildLocal.unity`
  VẪN GIỮ NGUYÊN** (object vẫn IS-A LocalMapHost). Đã verify: **8/8 tham chiếu `_bootstrap` sống sót,
  0 mất.** Đổi sang interface + field `MonoBehaviour` sẽ xoá sạch wiring của scene đang chạy tốt.
- Hợp đồng CHỈ 4 thành viên — đo từ nhu cầu thật của 14 file, không đoán: `Data` (8 file) ·
  `Grid` (3) · `Pathfinder` (3) · `OnGenerated` (6). **`OnGenerated` giữ nguyên dạng FIELD**
  (`Action<LocalMapData>`) chứ không đổi thành `event`, để 6 consumer đang `+=`/`-=` không phải sửa.
- **CỐ Ý KHÔNG kéo lên lớp cha** những thứ Wild-riêng: `SourceTile`/`WorldConfig`/`worldSeed`/
  `tileX`/`tileY`/`Regenerate`. 3 file cần chúng (`WildHUDController`, `BeastSpawner`,
  `WildCursorReadout`) vẫn khai kiểu `WildSceneBootstrap` như cũ. Đừng "cho đủ bộ".
- 9 file đã nới kiểu sang `LocalMapHost`; `CitySceneBootstrap` giờ **kế thừa `LocalMapHost`** (bỏ
  property `Grid` riêng — lớp cha đã có `Data?.Grid`, cùng instance).
- **City lần đầu có `RegionPathfinder` M14** (trước chỉ có `LocalPathfinder` A* cũ). Verify THẬT
  trong Play Mode: pathfind từ cổng thành (200,399) → cửa thành chủ (186,184) = **225 bước**.
- Verify: CityScene Play OK (Data/Grid 400×400/Pathfinder đều có) · **WildLocal Play OK, 55 FPS,
  0 lỗi** · test 20/20 + 16/16.
- **Số đo FPS củng cố chẩn đoán Individual-mode:** Wild 200×200 (40k ô) = **55 FPS**, City 400×400
  (160k ô) = **6 FPS**. Gấp 4 số ô → tụt ~9 lần. Không phải lỗi renderer nào cả.
- **CÒN LẠI:** B3 (`CityBuildingProvider`/`CityLotProvider` cắm `ITileInteractableProvider`) · B4
  (port taskbar/hub + 4 thứ City-only). `LocalMapData` vẫn nằm ở namespace `World.Wild.Generation` —
  đổi namespace là churn lớn, để riêng, không gộp vào B3/B4.

**B3 XONG (2026-08-01) — tương tác city + nội thất tái dùng `WildBuildingDefinition`:**
- **Câu hỏi user làm đổi hình B3:** *"player muốn tương tác với tượng / vật TRONG nhà thì sao?"*
  Bản phát biểu đầu của mình SAI ("1 provider trả công trình cho cả footprint" ⇒ ô sàn bị chiếm chỗ).
  Đọc code mới thấy kiến trúc đã lo sẵn: **`ITileInteractableProvider.Collect(... into)` là CỘNG
  DỒN**, `TileInteractableScanner.Scan(cell)` chạy MỌI provider rồi gộp ⇒ 1 ô ra NHIỀU mục
  ("Tửu Lầu" + "Bàn Ghế" + "Tượng"). Đó là lý do `WildContextMenu` là DANH SÁCH.
  **Test 2 của `M11CityInteractionTest` giữ bất biến này** — ai "tối ưu" thành first-wins thì
  right-click cái bàn trong quán sẽ ra cái quán, lỗi im lặng.
- `World/City/Interaction/CityBuildingProvider.cs` + `CityLotProvider.cs`. **Index `Dictionary<ô →
  vật>` dựng SẴN trong ctor** — quét tuyến tính 546 công trình × hàng trăm ô mỗi lần right-click là
  đủ gây khựng. `InteractableKind` += `CityBuilding = 12`, `CityLot = 13` (append-only).
- **CỐ Ý chưa gắn verb nào (Commands rỗng = chỉ xem info).** Verb chỉ chạy nếu
  `WildSceneInteraction.HandleAction` có nhánh; thêm "Enter"/"Trade"/"Claim" bây giờ = nút bấm vào
  không làm gì, tệ hơn không có nút.
- **Nội thất: TÁI DÙNG `WildBuildingDefinition`** (user chốt) — không đẻ kiểu thứ ba. 6 món mới trong
  `wild_buildings/`: `building_table_chairs` (bàn ghế, KHÔNG chức năng) · `building_potted_plant` ·
  `building_small_chest` (`core:storage` capacity 30) · `building_bed_simple` (`restEffectiveness` 2.0
  > lều trại 1.5) · `building_bookshelf` · `building_shop_counter`. Ăn sẵn provider + job + M26 +
  `JobGiver_Rest` mà không viết code.
- **Kiểm kê content trước khi lo "chưa đủ nội thất"**: đã có 9 wild_building (bench/smelter/tent/
  storage = nội thất CÓ CHỨC NĂNG) + `ancient_stele` (đúng "cái tượng"). **Art KHÔNG chặn**: chỉ
  3/21 feature có sprite, 0/9 building có icon — cả game đang chạy bằng ô màu fallback.
- Test: menu `Run M11 City Interaction Tests` (10 case) + 20/20 + 16/16, console sạch.
- **CÒN LẠI:** B3.5 (pass sinh nội thất vào nhà — bài toán GENERATION, cần *gán vai trò cho từng
  phòng* vì `BuildingInterior.Rooms` mới chỉ là danh sách ô, và luật CHỪA LỐI ĐI vì
  `MapFeature.BlocksTile` → `SetPassable(false)`, đặt tượng chắn cửa = tự nhốt phòng) · B4 (port
  taskbar/hub + 4 thứ City-only).

**B3.5 XONG (2026-08-01) — bày nội thất vào nhà thành phố:**
- `World/City/Generation/CityFurnisher.cs` + `CityFurnishingRule` (JSON `Data/city_furnishing/`,
  6 luật theo ROLE công trình). Nội thất = `WildBuilding` trạng thái `Built` trong
  `LocalMapData.WildBuildings` ⇒ ăn sẵn `WildBuildingProvider` (right-click),
  `BuildingFunctionRegistry` (rương → **kho THẬT**), `JobGiver_Rest` (giường qua `restEffectiveness`).
  **0 dòng code hiển thị/tương tác mới.** Đo: **3.767 món / 546 nhà, 1.380 kho thật.**
- **Bày theo ROLE công trình, KHÔNG theo vai trò từng phòng** — `BuildingInterior.Rooms` mới chỉ là
  danh sách Ô, chưa biết "phòng này là bếp hay phòng ngủ". Gán vai trò phòng là bài toán riêng; bày
  theo role đã đủ để nhà có đồ mà không phải bịa thêm một tầng khái niệm.
- **2 bất biến, kiểm SAU MỖI lần đặt thử (không phải "né ô hiểm"):** ① không đặt lên ô CỬA ② BFS lại
  từ cửa, mọi ô sàn còn lại phải tới được. Bắt buộc vì `WildBuilding` Built **chặn mọi ô footprint**
  (luật M17) — một cái tủ đặt sai đủ cắt phòng làm đôi. Thêm trần `maxFillPercent` (25-35%).
- **⚠ BUG GỐC LỘ RA NHỜ B3.5 — hình `twin` sinh nhà KHÔNG LIÊN THÔNG:** cầu nối giữa 2 khối chỉ rộng
  **2 ô**, bọc tường xong thì CẢ HAI ô đều nằm trên đường viền → cầu không còn ô ruột → hai nửa nhà
  tách hẳn. `RepairConnectivity` cũng bó tay vì nó cố ý không đục ô viền. Sửa: **cầu rộng ≥ 3** (ô
  giữa có đủ 4 hàng xóm trong nhà nên là ô ruột thật).
- **BÀI HỌC QUAN TRỌNG HƠN CẢ BUG:** lỗi lọt qua CẢ `M11InteriorTest` LẪN `M11CityTest` vì **cả hai
  chạy trên bộ def TỔNG HỢP tự khai trong file test**, mà `twin` chỉ có trong JSON thật. Test dùng
  data giả thì chỉ canh được data giả. Đã thêm `M11CityInteractionTest` Test 6 — chạy trên **JSON
  THẬT**, đòi mọi công trình có ruột đều liên thông (546 căn).
- Nối vào `CitySceneBootstrap` **TRƯỚC khi dựng `RegionPathfinder`** (đồ đạc chặn ô, pathfinder phải
  thấy trạng thái cuối).
- Test: `Run M11 City Interaction Tests` **16/16** (10 case B3 + 6 case B3.5) · 20/20 · 16/16.

**⚠ DÍNH LẠI BẪY "CODE XONG NHƯNG CHƯA GẮN VÀO SCENE" — LẦN THỨ 3 CỦA DỰ ÁN (2026-08-01):**
Mình báo "B3/B3.5 xong" trong khi **người chơi không chạm được gì**: user phải hỏi *"cậu đã thêm nó
vào game chưa để mình chơi thử"* mới lộ. Trạng thái thật lúc đó:
- Nội thất SINH RA thật trong CityScene (log `Đã bày 3767 món`) nhưng **VÔ HÌNH** —
  `CityTilemapRenderer` không đọc `LocalMapData`, mà nội thất sống ở đó chứ không ở `CityLayout`.
- `CityBuildingProvider`/`CityLotProvider` **chỉ được dùng trong file TEST**, không scene nào đăng ký;
  CityScene không có `WildSceneInteraction`/`WildContextMenu`/`WildInfoPanel` ⇒ **không right-click được**.
**Xong-với-test ≠ xong-với-người-chơi.** Từ nay mỗi bước phải tự trả lời "player chạm được chưa" TRƯỚC
khi báo xong, đúng tinh thần cảnh báo đã có sẵn ở CLAUDE.md (đã dính với `ForestLifecycleSystem`
2026-07-27 và `DayNightController` 2026-07-28).
**Đã sửa phần NHÌN THẤY ngay:** `WildBuildingDefinition` += `debugColor` (cùng quy ước
`terrain_tiles`/`CityLotDefinition`; đã set cho cả 15 building) · `CityTilemapRenderer.Render` nhận
thêm tham số `LocalMapData` và vẽ `PaintFurniture` · `CityHUDController` truyền `_bootstrap.Data`
xuống. Verify Play Mode thật: nội thất hiện trong nhà, zoom sát 181 FPS.
**Phần TƯƠNG TÁC (right-click) vẫn CHƯA — đó chính là B4.**

**BẢNG QUY MÔ ĐỢT 1 (≈70 ô/nhà, nhà 6×5 — số này đã LỖI THỜI sau đợt 2, giữ để tham chiếu cách tính):**
96 → ~55 căn (làng) · 128 → ~125 · **160 → ~200 (default mới)** · 200 → ~310 · 256 → ~510 (preset
`large_city`, hạ từ 400) · 400 → ~1400 (KHÔNG dùng). Sàn cứng ≈ 120 (đủ 2 vành block quanh castle).
Đo thật ở 160×160: 177 công trình, ~40% diện tích nội thành là nhà, ~42% là đường, ~19% đất trống.

**Tham số phải giãn theo footprint mới** (`blockMaxSize` 16→24, `blockMinSize` 5→8, `alleyDepthMax` 10→13
= 2× chiều sâu nhà +1, `compoundMinDim` 6→**8** (để 6 thì NHÀ DÂN ăn trọn 1 lô), `buildingGap` 1→0,
`vacancyChance` .12→.08). **Footprint JSON nhà dãy 4×2/4×4 → 6×5/6×6** (19 file × 2 thư mục) — 4×2 bọc
tường thì ruột = 0, không bao giờ sinh được nội thất; `M11CityTest.MakeDefaultDefinitions` cũng phải sửa
theo không thì test không còn phản ánh data thật.

**CHƯA LÀM (biết rõ, không phải bỏ sót):** ① save — city vẫn lưu seed+tier+delta và tái sinh layout từ
seed; nội thất deterministic nên vẫn đúng, NHƯNG đồ đạc/nội thất player đặt sau này thì không tái sinh
được, phải chốt trước khi làm tiếp (user chủ động hoãn). ② City scene KHÔNG có runtime sim: không
`LocalMapData`, không `PawnAiBrain`, pathfinding vẫn `LocalPathfinder` cũ (M14 `RegionPathfinder` chỉ Wild
dùng) → chưa ai ở trong những căn nhà đó. Hướng rẻ nhất đã phân tích: cho city gen ghi ra `LocalMapData`
(lớp đó không có gì Wild-specific, chỉ nằm nhầm namespace) rồi dùng nguyên runtime Wild, thay vì port từng
hệ sang City scene. ③ Chưa có hệ MÁI (RimWorld ẩn mái khi vào trong) — hiện nhìn xuyên thấy ruột nhà.
④ `streetWidth` vẫn 4 (chiếm 42% nội thành) — hạ xuống 3 được thêm ~10 nhà, nhưng đó là quyết định thẩm mỹ
chưa hỏi user.

**Scene:** `CityScene.unity` `gridWidth/Height` 400 → **0** (dùng preset large_city 256) — đã lưu qua MCP.

Liên quan: [[project-citygen-wallcity]], [[project-nizhangjie]], [[project-global-structure-hub]],
[[project-region-pathfinding]].
