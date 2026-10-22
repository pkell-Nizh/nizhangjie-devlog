---
name: project-hud-prefab-unify
description: "Sự cố HUD.prefab dùng chung Wild+City bị Apply All/Revert đè lẫn nhau + phục hồi thành hợp nhất có chủ đích (pawn giờ 'sống' ở cả City); ĐÃ VERIFY Play Mode 2026-08-18"
metadata: 
  node_type: memory
  type: project
  modified: 2026-08-27T12:26:47.333Z
  originSessionId: 40731056-8a28-466c-a15e-10c72c682925
---

## Sự cố

User bấm **Apply All** trên HUD instance ở CityScene rồi **Revert** HUD instance ở WildLocal. `HUD` là
**1 prefab DÙNG CHUNG** (`Assets/Prefabs/UI/GobalUI/Gobal_use/HUD.prefab`) cho cả 2 scene. Apply All đẩy
TOÀN BỘ khác biệt của City vào prefab gốc; Revert kéo Wild về khớp đúng file đó — Wild mất sạch phần
riêng của mình, dính theo phần chỉ City mới có.

**Không có git** (project không phải repo), không OneDrive, không File History, không Shadow Copy
(kiểm tra hết). **Undo của Editor cũng cạn** — chuyển scene (cả do user thao tác tay lẫn do
`manage_scene load` của MCP) xoá sạch Undo stack. Không có đường phục hồi tự động nào.

## Quyết định của user: không phục hồi trạng thái phân kỳ cũ, HỢP NHẤT có chủ đích

> "code và md không hề biến mất, nhất là đây cũng là cơ hội để đồng bộ của wildscene và cityscene, vì trên
> bản chất cityscene và wildscene là một, chúng chỉ có các bộ nạp dữ liệu khởi tạo khác nhau, giống như các
> chunks trong minecraft vậy"

Sau đó chốt thêm: *"lúc trước chúng ta không hề có ý định cho pawn sống trong cityscene nhưng giờ thì khác
rồi, thứ cần thay đổi bây giờ lại là CityScene"* — tức: KHÔNG dựng lại y hệt bản Wild cũ, mà sửa **City**
để bắt kịp giả định mới (pawn sống đầy đủ ở cả 2 scene, không riêng Wild).

## Nguyên tắc rút ra — KHÔNG PHẢI mọi thứ nên hợp nhất

`CityCursorReadout.cs` tự có sẵn dòng giải thích: dự án **đã từng thử tái dùng `WildCursorReadout` ở City
rồi phát hiện phải tách ra** (2026-08-02) — nó khai `_renderer` kiểu `WildTilemapRenderer`, City dùng
`CityTilemapRenderer`, 2 type không gán được cho nhau, field trống lặng câm. Đây là khác biệt **CÓ CHỦ Ý**
(loại "bộ nạp/renderer" thật sự khác — đúng nghĩa "chunk khác loader" mà user nói), không phải phân kỳ
tình cờ. Bài học: **trước khi hợp nhất bất cứ thứ gì bị corrupt, phải hỏi "khác biệt này có LÝ DO KỸ THUẬT
THẬT hay chỉ là tình cờ chưa ai dọn?"** — câu trả lời quyết định sửa ở prefab gốc (hợp nhất) hay sửa
per-instance override (giữ tách).

Áp dụng: `CursorReadout` → sửa lại `WildCursorReadout` **CHỈ trong Wild** (per-instance, City giữ
`CityCursorReadout`). `HealthAlertBar`/`HealthPanel` → đưa vào **prefab gốc** (dùng chung), vì lý do vắng
mặt ở City chỉ là "trước đây không định cho pawn sống ở City" — giả định đã hết hạn.

## Đã làm

1. **`CitySceneBootstrap.EnsureMapSystems()`** — thêm `WildHealthSystem` + `WildHealthAlertBar` (mirror
   cách `PawnNeedsSystem`/`PawnAiBrain` đã được thêm trước đó cùng lý do "tổng quát, quét
   `WildPlayerToken.All`/`FindObjectsByType`, không đọc field Wild-riêng"). Đọc code xác nhận TRƯỚC khi
   gắn: `WildHealthSystem.TickBeasts` guard `FindFirstObjectByType<BeastSpawner>() == null → return` nên
   an toàn tuyệt đối ở City (không có BeastSpawner).
2. **Mở `HUD.prefab` qua Prefab Stage** (`open_prefab_stage`/`save_prefab_stage`/`close_prefab_stage`) —
   ĐÚNG đường sửa prefab gốc, KHÔNG lặp lại lỗi "sửa instance rồi Apply" đã gây sự cố này.
   - Dựng lại `HealthAlertBar` (đơn giản — 1 TMP_Text + `WildHealthAlertBar`, tự tìm `_label` qua
     `GetComponent` nên không cần wire tay).
   - Dựng lại `HealthPanel` (`WildHealthPanel`, ~20 SerializeField) — giữ NGUYÊN class 398 dòng đã có sẵn
     (không viết lại logic), chỉ dựng UI hierarchy rồi wire field qua `SerializedObject` (kỹ thuật giống
     hệt lúc wire nút taskbar trước đó — `UnityEventTools.AddVoidPersistentListener` cho 4 nút, set
     `objectReferenceValue` cho các field tham chiếu). Dùng ĐÚNG hằng số cột đã đọc trong
     `WildHealthRow.cs` (ColumnWidths 160/85/105/95/140, spacing 4px, cao 26px) — không đoán.
   - `_popupMenu` không cần wire tay — tự `FindFirstObjectByType<WildBuildMaterialMenu>` nếu null.
3. **Sửa `CursorReadout` ở Wild** (per-instance, KHÔNG đụng prefab) — gỡ `CityCursorReadout`, thêm
   `WildCursorReadout`, wire `_bootstrap`/`_renderer` qua `SerializedObject` (2 field KHÔNG có fallback
   tự tìm, khác `_camera`/`_label` có `Camera.main`/`GetComponent` fallback).

## ⚠ Bẫy MCP gặp trong đợt phục hồi

- `manage_prefabs get_hierarchy` trả **783 object, 332KB** — vượt giới hạn token, phải đọc qua file +
  PowerShell `ConvertFrom-Json` lọc theo tên (Bash không có `jq`/`python3` trên máy này).
- `find_gameobjects by_name` **bỏ sót object có tên kèm khoảng trắng cuối** (`"CursorReadout "`) — search
  "Cursor"/"CursorReadout" đều ra rỗng dù object tồn tại và active. Phải lấy instanceID qua
  `get_hierarchy` (liệt kê đầy đủ) rồi thao tác thẳng bằng ID, đừng tin `find_gameobjects` cho tên có
  khoảng trắng/ký tự lạ.
- `execute_code` chạy bằng CodeDom (C#6) không phải Roslyn mặc định — **không hỗ trợ local function**
  (`void SetRef(...) {...}` khai giữa thân code) — phải viết bằng mảng + vòng lặp `for`.
- Việc gán prefab con mới không tự kích `refresh_triggered` — nhưng lần này không cần vì thao tác nằm
  trong Prefab Stage, `save_prefab_stage` tự flush.

## ĐÃ VERIFY Play Mode — cả 2 scene, dữ liệu thương tích THẬT (không đọc code suông)

**Wild**: tạo vết "Bầm Tím" (sev 0.6) qua `HealthRegistry.AddInjury` thật → `VitalsLabel` = "Máu 121/121 ·
Đau 22%" (đúng `HealthOps`) → bấm dòng row thật → `DetailRoot` mở, `DetailSpec`/`DetailList`/`DetailOptions`
đúng nội dung InjuryDef → nút Priority/NoTend/Herb **tự vô hiệu đúng** (vết này không cần băng — khớp luật
`editable = !Tended && def.needsTending`). Thêm vết chảy máu → `AlertBar` vẫn rỗng vì pawn **chưa gục**
(`RemainingDeathHours` = Infinity khi `!downed`) — đúng thiết kế gốc, không phải lỗi bản dựng lại.

**City** (lần đầu tiên có hệ này): `WildHealthSystem` tự gắn ✓, `HealthAlertBar` xuất hiện đúng 1 bản
(không gắn trùng nhờ guard `FindObjectsInactive.Include`) ✓, `CityCursorReadout` còn nguyên/không bị đụng
✓, tạo vết thương thật → panel mở tại City → "Máu 165/165 · Đau 18%" — **chạy đúng y hệt Wild**.

Test suite: Decoupling 42/0, Companion 49/0 (chạy 2 lần, cả sau đợt sửa Wild lẫn sau đợt City).

## ⚠⚠ Hư hỏng NẶNG NHẤT — phát hiện SAU khi tưởng đã xong (đợt 2, cùng ngày)

User báo: vào Wild map trắng trơn (chỉ nền màu phẳng), thấy beast nhưng KHÔNG thấy pawn/tile nào. Đây mới
là hư hỏng NGHIÊM TRỌNG NHẤT của sự cố Apply All/Revert — lớn hơn hẳn Health UI, và suýt bị bỏ sót vì lúc
soát HUD trước đó chỉ kiểm panel/nút, không kiểm GameObject `HUD` **tự nó có những component gì**.

**Gốc bệnh**: `HUD` (GameObject root) mang **1 component orchestrator gắn RIÊNG từng scene** —
`WildHUDController` ở Wild, `CityCharacterSheetPanel` ở City (per-instance, không phải phần prefab
chung). Apply All từ City đẩy `CityCharacterSheetPanel` vào prefab gốc; Revert ở Wild xoá sạch
`WildHUDController` (không có trong prefab đã bị City hoá) và thay bằng `CityCharacterSheetPanel`.

`WildHUDController.Awake()` subscribe `WildSceneBootstrap.OnGenerated` — KHÔNG còn tồn tại thì
**KHÔNG AI GỌI `_renderer.Render(data)` LẪN `_token.Init(...)`** — giải thích ĐÚNG cả 2 triệu chứng cùng
lúc (tile không vẽ, pawn không init) mà console 0 lỗi (guard `if (_bootstrap != null)` nuốt im lặng).
Quái vẫn hiện vì `BeastSpawner` tự subscribe `OnGenerated` RIÊNG, không phụ thuộc `WildHUDController`.

**Bài học lớn nhất của cả sự cố**: `CursorReadout`/`WildHUDController` đều là **field tham chiếu CHÉO
sang object khác trong CHÍNH SCENE đó** (không nằm trong prefab) — Apply All/Revert xoá SẠCH mọi tham
chiếu chéo kiểu này mà không báo gì. Kiểm 1-2 component không đủ; phải nghi ngờ **MỌI** component gắn
trên prefab instance có field trỏ ra ngoài prefab.

**Cách chẩn đoán dùng lại được**: `Tilemap.cellBounds.size == (0,0,1)` = CHƯA từng có ô nào được vẽ (khác
hẳn "vẽ rồi nhưng sai gì đó"). Từ đó truy ngược ai lẽ ra phải gọi `.Render()` — không phải sửa renderer,
mà là tìm orchestrator bị mất.

**Vá**: gỡ `CityCharacterSheetPanel` khỏi `HUD` (Wild), thêm lại `WildHUDController`, wire 5 field
(`_bootstrap`/`_renderer`/`_token`/`_camera`/`_title`) qua `SerializedObject` — cùng kỹ thuật đã dùng cho
`CursorReadout`. Tiện thể phát hiện thêm 1 mảnh vỡ: `HUD/CharacterSheet` (panel City, `childCount=4`)
**active mặc định** ở Wild (khác các panel khác đều tắt sẵn) — che kín màn hình. Tắt nó ở Wild (giữ
nguyên active ở City — đúng ý City, panel/component đó vẫn thuộc về City).

**Verify Play Mode + ảnh chụp thật** (`manage_camera screenshot`): trước vá — panel "Hồ Sơ Nhân Vật" che
kín màn hình, không tile. Sau vá — tile hiện đủ (`cellBounds` 200×200), **4 pawn** đều có vị trí thật
(avatar + 3 party test), 0 quái (đúng, map hiện tại không có ổ). City verify riêng: `CityCharacterSheetPanel`
còn nguyên, không dính lỗi này (nó là NGUỒN của Apply All nên không bị Revert đụng tới); ảnh chụp xác nhận
road/terrain/minimap/pawn đều đúng. Test 2 lần (Wild + City): Decoupling 42/0, Companion 49/0.

## ⚠ Bẫy `manage_gameobject create`: tự "bù trừ" scale, đôi khi tính ra 0

`HUD` (Canvas, ScreenSpaceOverlay) có `localScale = 0.57` — giá trị BASELINE có từ trước, không phải do
đợt sửa này. Mọi object CÓ SẴN từ trước (`TimeBar`, `Button` gốc trong Taskbar...) đều giữ
`localScale = 1` và ăn theo `lossyScale = 0.57` của cha — đúng quy ước UI (đừng đụng scale, chỉnh
`sizeDelta`).

**`manage_gameobject create` (KHÔNG phải `duplicate`) tự tính bù `localScale` để ép `lossyScale` con về
đúng 1.0** — với `HealthPanel` ra `1.74` (≈ 1/0.57, đúng công thức bù, chỉ là SAI QUY ƯỚC vì không ai
khác trong HUD làm vậy nên `HealthPanel` bị to hơn hẳn phần còn lại ~1.75 lần). Với `HealthAlertBar` phép
tính ra thẳng **`(0,0,0)` — VÔ HÌNH HOÀN TOÀN**, khả năng do `Screen.width/height` đọc được 0 tại đúng
khoảnh khắc tạo (Game view chưa focus/chưa render). Không có warning, không có lỗi — chỉ lặng lẽ vẽ ra
một object 0×0×0.

**Object tạo bằng `duplicate`** (như `RosterButton`, nhân bản từ `ScheduleButton`/`SettingsButton`) **KHÔNG
dính** — Duplicate chỉ chép transform gốc, không tự "bù" gì cả. Đã kiểm cả 2 scene, `RosterButton` luôn
`localScale = 1`.

**Quy tắc rút ra cho lần sau**: sau MỌI lần dùng `manage_gameobject create` bên trong 1 Canvas/prefab đã
có scale gốc khác 1 — **luôn đọc lại `localScale` của object vừa tạo, đừng mặc định là 1**. Không đủ để
kiểm mỗi field/vị trí — cái làm UI biến mất hoàn toàn (`0,0,0`) lại là thứ dễ bị bỏ sót nhất vì không có
lỗi nào báo, chỉ là "trông như trống".

**Vá**: mở lại Prefab Stage, set thẳng `HealthAlertBar.localScale`/`HealthPanel.localScale` về `(1,1,1)`
(con cháu bên trong 2 object này đã đúng `1` từ đầu, không cần sửa gì thêm — bug chỉ xảy ra ở 2 object
GỐC được tạo trực tiếp dưới `HUD`). Verify lại bằng ảnh chụp Play Mode: bảng Thương Tích hiện đúng tỉ lệ,
đọc được chữ, không còn phóng to lệch so với phần còn lại của HUD. Test: Decoupling 42/0, Companion 49/0.

## Chưa hoàn hảo (ghi rõ, không giấu)

- Layout của `HealthPanel` dựng lại là CHỨC NĂNG ĐÚNG, KHÔNG pixel-perfect như bản gốc đã mất (không có
  tham chiếu để so khớp màu/khoảng cách/vị trí chính xác). Panel cao 560, vùng Content+Detail có thể tràn
  nhẹ ra ngoài khung nếu số vết thương nhiều — chưa test trường hợp >3 vết cùng lúc.
- 6 panel City (Minimap/BuildingInfo/JobDetail/CityHub/CharacterSheet/Bank) hiện **VẪN nằm trong prefab
  chung**, tức hiện diện (vô hại, đã verify 0 lỗi/warning) nhưng vô nghĩa ở Wild. Chưa quyết định có nên
  dọn khỏi Wild hay để nguyên theo tinh thần "chunk" (data quyết định có gì hiển thị, không phải scene).

Liên quan: [[project-m30-health]] (hệ gốc bị ảnh hưởng), [[project-nizhangjie_gotchas]] (bẫy MCP mới nên
thêm vào).

## M56q (2026-08-27) — dư chấn CUỐI CÙNG của sự cố này lộ ra, đã vá

Khi sửa lại `CityCharacterSheetPanel` (đổi cột "KHÁNG TÍNH" → "TAY NGHỀ" theo phản hồi user), soát GameObject
`HUD/CharacterSheet` thì thấy nó mang **2 component `CityCharacterSheetPanel` cùng lúc** (cả 2 wire y hệt
nhau) — gần như chắc chắn là tàn dư CHÍNH XÁC của sự cố Apply All/Revert ở trên (1 bản gốc + 1 bản bị đẩy
lại lúc Apply All), nằm im từ đó tới giờ vì không ai soát `GetComponents<T>().Length` cho panel này. Đã
xoá bản thừa (`manage_components remove` với `component_index=1`, giữ index 0). Không phát hiện triệu
chứng lạ nào khác từ nó trước khi vá (Unity không lỗi, chỉ 2 component song song đều đọc/ghi đúng field,
lãng phí nhẹ chứ không sai logic) — nhưng dọn cho sạch vì đây đúng loại rác "không lỗi gì nên không ai
để ý" mà luật C1 cảnh báo.

Đồng thời tái cấu trúc **cả 4 cột nội dung** (`ColDetails`/`ColIdentity`/`ColCombat`/`ColCapabilities`,
đổi tên từ `ColResist`) thành `ScrollRect + Viewport(RectMask2D) + Content(VerticalLayoutGroup+
ContentSizeFitter)` — script chỉ cần trỏ field vào `Content`, không đổi gì khác. Verify Play Mode: nhồi 30
dòng giả vào 1 cột → `Content` cao 918 > `Viewport` 490 → `ScrollRect.verticalNormalizedPosition=0` di
chuyển `Content.anchoredPosition.y` đúng hướng — cuộn thật chạy được, không chỉ dựng khung rỗng.

Xem [[project_m56n_race]] cho phần "TAY NGHỀ" đọc từ đúng hệ Capability/Race gatekeeper M56n-p.
