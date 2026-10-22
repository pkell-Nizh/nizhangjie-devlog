---
name: project_m61_tang3_build_persist
description: "M61 — vá lỗ hổng Tầng 3: building/tường/kho/trạm/vật rơi (material) persist thật theo ô world, cả RAM lẫn save file"
metadata:
  node_type: memory
  type: project
  originSessionId: 05b3bdd6-6055-48c2-91c4-abfe02c1257a
  modified: 2026-08-28T14:42:51.661Z
---

M61 (2026-08-28) — user hỏi thẳng "cái gì lỗ hổng cơ" rồi chốt "vá luôn lỗ hổng Tầng 3 đi": trước M61,
`WildSceneBootstrap.Generate()` tái sinh `LocalMapData` HOÀN TOÀN MỚI mỗi lần vào lại 1 ô Wild — mọi
building/tường/kho/trạm player dựng biến mất im lặng (terrain tất định từ seed nên NHÌN như "map cũ",
nhưng nhà thì mất thật). Khác bed OWNERSHIP (M60) — cái đó sống được nhờ tách hẳn sang
`BedClaimRegistry`, không phải vì Tầng 3 đã xong. Xem [[project_m60_bed]] mục "Nợ còn lại" (đã sửa lại).

## Kiến trúc — mirror WildMapPawnState (Tầng 1) + WorldEventStore, KHÔNG đẻ hệ song song

`World/Wild/Persistence/WildMapBuildState.cs` — POCO `[Serializable]`, `tileKey` (dùng LẠI ĐÚNG công thức
`WildMapPawnState.TileKey` — 3 sổ Tầng 1/bed/build nói CHUNG một khoá cho cùng 1 ô) + 6 `List<T>` snapshot
struct phẳng (BuildPiece/WildBuilding/Storage/Workstation/Producer/GroundItem-material) — flat list chứ
KHÔNG Dictionary (bẫy JsonUtility quen thuộc, mirror `WorldEventTileState`/M40 `WildQuestSeedDefinition`).

`World/Wild/Persistence/WildMapBuildStateStore.cs` — MonoBehaviour DontDestroyOnLoad (mirror
`WildMapPawnState`), `Capture(key, LocalMapData)`/`Restore(key, LocalMapData)` là 2 CỬA DUY NHẤT đọc/ghi
6 danh sách của `LocalMapData`.

**Footprint building = HÌNH CHỮ NHẬT origin+width+height, KHÔNG phải list cell** — verify được bằng đọc
code (`WildBuilding`/`WildStorageInstance`/`WildWorkstation`/`WildProducer` đều dựng footprint kiểu
`for x for y` từ 1 origin) nên nén được, `RectCells` dựng lại ĐÚNG thứ tự gốc (`Cells[0]`/`Origin` luôn là
góc dưới-trái, khớp semantics các lớp gốc).

## Chỗ cắm vào pipeline — bắt buộc TRƯỚC pathfinder, KHÔNG qua OnGenerated

`WildSceneBootstrap.FinishGeneration` thêm bước 3b (`WildMapBuildStateStore.Restore`) — **NGAY SAU** feature-
blocking passability (bước 3), **NGAY TRƯỚC** dựng `RegionPathfinder` (bước 4). Thử qua sự kiện
`OnGenerated` trước — SAI, event đó bắn SAU khi pathfinder đã dựng xong nên building khôi phục lại sẽ
không chặn đúng ô trong PathGrid (pawn đi xuyên tường vừa hiện ra lại). Đây là lý do M61 phải sửa TRỰC
TIẾP `WildSceneBootstrap.cs` thay vì tự gắn qua `EnsureMapSystems()` như M58/M59.

Restore **CỐ Ý KHÔNG gọi `BuildingFunctionRegistry.ActivateAll`** — snapshot đã mang đúng nội dung THẬT
của Storage/Workstation/Producer, activate lại sẽ dựng bản SONG SONG rỗng đè lên.

`ReturnToWorldMapButton.CaptureWildStateIfAny` (đổi tên từ `CaptureWildPositionsIfAny`, M58) giờ chụp CẢ
Tầng 1 (vị trí pawn) LẪN Tầng 3 (building) trong cùng 1 lần rời map — **loại trừ map vẽ tay** (`bootstrap.
CellIndex >= 0`, M33) vì bí cảnh không có khái niệm "ô world" để mà khôi phục kiểu này, nội dung của nó
do người viết đặt sẵn qua `FixedMapLoader`.

## Scope — CHỈ 6 loại, nói thẳng cái gì KHÔNG nằm trong lần vá này

Building/tường (Built + Blueprint, kể cả tiến độ xây dở) · WildBuilding (kèm `OwnerRef` của M60, khớp cả
2 sổ) · Storage (kèm NỘI DUNG túi qua `PlayerItemBag.LoadFrom/ToList` có sẵn từ M18, không cần viết mới) ·
Workstation (kèm TOÀN BỘ WorkOrder — recipe/mode/remaining/suspended/progress) · Producer (kèm lịch
`NextProduceTick`) · GroundItem loại MATERIAL.

**CHƯA phủ (cắt scope có chủ đích, không phải quên):**
- Zone (vùng player vẽ) và Designation (đánh dấu việc ngắn hạn) — cả hai đổi liên tục, chi phí/lợi ích
  khác hẳn "nhà đứng yên hàng chục giờ".
- Feature tự nhiên đã khai thác (quặng/cây đã đốn) RESET về "còn nguyên" mỗi lần vào lại — lỗ nhỏ hơn hẳn
  "cả căn nhà biến mất", để lại cho đợt sau nếu user thấy cần.
- GroundItem loại EQUIPMENT (`WorldItemDrop.IsEquipment==true`) — rớt xuống đất bị mất khi rời map, dù
  material thì còn.

## Bẫy tự dính khi wire SaveSystem (đã sửa, không phải hỏi user)

**"Bấm Lưu MÀ KHÔNG rời map trước" sẽ lưu bản CŨ (hoặc rỗng) của chính ô đang đứng** — `WildMapBuildStateStore.
Capture` chỉ chạy lúc RỜI map (`ReturnToWorldMapButton`), nên gọi `SaveManager.Save()` khi đang đứng giữa
map xây dở sẽ ghi xuống đĩa snapshot của LẦN RỜI TRƯỚC (hoặc không có gì nếu đây là lần đầu vào ô này) —
đúng lớp bug mà M58 (Tầng 1, vị trí pawn) NGẦM ĐỊNH có nhưng chưa ai đo tới. Vá bằng
`WildMapBuildStateStore.CaptureCurrentSceneIfLoaded()` (static, mirror logic của
`ReturnToWorldMapButton`) gọi Ở ĐẦU `SaveManager.WriteFile()` — mọi đường lưu (Save/Create) đều tự chụp
lại ô ĐANG SỐNG trước khi ghi. **Tầng 1 (vị trí pawn) CHƯA được vá tương tự** — cùng lớp lỗi, để nguyên vì
nằm ngoài phạm vi "Tầng 3" user yêu cầu, ghi lại đây để không quên.

## Verify

Test menu `Run BuildPersist Tests` (**31 case, mới**) — round-trip riêng từng loại (6 loại + IsEmpty +
ReplaceAll) + round-trip QUA `JsonUtility.ToJson/FromJson` THẬT (mirror kỷ luật M53: lớp lỗi hay gặp nhất
của save không phải logic mà là "JsonUtility lặng lẽ không serialize được kiểu này"). Hồi quy: Bed 24/24,
WildMapPawnState 19/19, AbsentPartyNeeds 18/18, Party 50/50, FixedMap 37/37, BuildPiece 10/10,
JobGiverConstruction 6/6, WildBuilding 15/15, Production 39/39 — xác nhận đổi chữ ký
`FinishGeneration` không vỡ gì.

**Play Mode thật (2026-08-28), ĐÚNG production path đầu-cuối, KHÔNG gọi tắt:**
1. Đặt tường Built (chặn ô) + cửa Built (không chặn) + `WildBuilding` giường Built có `OwnerRef` + kho có
   nội dung (25 gỗ + 6 thịt) + trạm chế tác có 1 đơn (4 lượt, dở 2.5s) + 1 vật rơi material lên map THẬT
   (seed=1337, tile 5,7) → bấm **ĐÚNG nút "Rời Vùng"** (`ReturnToWorldMapButton.ReturnToWorldMap()`) →
   xác nhận `WildMapBuildStateStore` đã có đúng key với đủ 5/6 loại (0 ground-equipment vì không test loại
   đó ở đây) → **ĐÚNG** `WorldMapHUD.OnEnter()` (qua `Show()` chọn lại tile 5,7) vào lại map → mọi thứ khôi
   phục ĐÚNG: tường vẫn chặn (`IsPassable=false`), cửa vẫn đi được, giường giữ nguyên `OwnerRef`, kho giữ
   nguyên 25/6, trạm giữ nguyên đơn+tiến độ+remaining, vật rơi giữ nguyên.
2. **Save-file thật trên đĩa** (mirror M53): đứng NGAY TRÊN map xây dở (chưa rời) → `SaveManager.Instance.
   Create(save)` → đọc thẳng file JSON trên `Application.persistentDataPath` → có đủ `wildTileBuilds` +
   `OwnerRef` + tên kho + tên công thức — xác nhận CHÍNH XÁC nhánh `CaptureCurrentSceneIfLoaded` vừa vá.
3. **"Tắt game mở lại" thật** (đúng lời user): `DestroyImmediate` HẲN GameObject
   `WildMapBuildStateStore` (RAM sạch tuyệt đối, không phải xoá nội dung) → `SaveManager.Instance.
   Load(saveId)` → store TỰ TÁI TẠO qua `FindOrCreate()` + nạp đúng nội dung từ đĩa → sinh lại map ô đó
   (`bootstrap.Generate()`) → building/kho/trạm hiện lại ĐẦY ĐỦ, `OwnerRef` khớp nguyên.
4. Dọn: xoá save test, `manage_editor stop` (Unity tự revert mọi thay đổi scene lúc Play Mode).

⚠ **Bẫy tự dính lúc verify (môi trường, không phải bug code — đã ghi nhiều lần các đợt trước):**
`EditorApplication.isPlaying` mới bật, `Time.frameCount` đứng ở 2 dù `realtimeSinceStartup` trôi (cùng họ
"Editor Play Mode không tick khi mất focus trong phiên MCP" đã ghi ở M36/M58/M59/M60) ⇒
`WorldMapSceneBootstrap.Map`/`WildSceneBootstrap.Data` còn null vì `Start()` (gọi `Generate()`) chưa từng
chạy. Vá bằng gọi THẲNG `bootstrap.Generate()` (public) thay vì chờ frame — **và phải tự set lại
`seedOverride` về đúng seed cũ (1337) trước khi gọi trên `WorldMapSceneBootstrap`**, vì gọi `Generate()`
lần đầu (không override) đọc seed RANDOM MỚI từ `WorldDataLoader.LoadConfig()` — tự tạo một thế giới khác
hẳn nếu không cẩn thận, khiến tile (5,7) trỏ sang biome/dữ liệu khác.

## M61b — vá nốt Tầng 1 (vị trí pawn) vào SaveSystem (2026-08-28, cùng ngày, user yêu cầu tiếp)

Ngay sau khi báo cáo M61 xong, user chốt luôn: "vá luôn Tầng 1 (vị trí pawn) đi". `WildMapPawnState`
([[project_m58_wildmapstate]]) trước đó CHỈ RAM (DontDestroyOnLoad) — sống được qua rời/vào map TRONG
CÙNG PHIÊN nhưng tắt game mở lại là mất, token về lại điểm vào. Vá ĐỐI XỨNG với M61 (building), dùng LẠI
đúng pattern:

- `WildMapPawnState` thêm `WildMapPawnTileState` (flat POCO tự mang `tileKey`, mirror `WildMapBuildState`)
  + `All`/`ReplaceAll` (đọc/ghi toàn sổ) + `CaptureCurrentSceneIfLoaded()` (static, mirror
  `WildMapBuildStateStore.CaptureCurrentSceneIfLoaded` — vá ĐÚNG cùng lỗ hổng "bấm Lưu mà không rời map
  trước" mà M61 vừa vá cho building, nay áp luôn cho vị trí pawn).
- `SaveData.wildTilePawns` (field mới) + `SaveSystem/WildMapPawnStateIO.cs` (mới, mirror
  `WildMapBuildStateIO` từng nét) + cắm vào cả 3 đường `SaveManager.Load/Create/WriteFile` — `WriteFile`
  gọi `CaptureCurrentSceneIfLoaded()` NGAY TRƯỚC `Capture(data)`, cùng chỗ với dòng tương đương của Tầng 3.
- **Khác Tầng 3 ở đúng 1 điểm:** vị trí pawn áp dụng cho CẢ map sinh ngẫu nhiên LẪN map vẽ tay (M33) —
  không cần guard `CellIndex >= 0` như building, vì `ReturnToWorldMapButton.CaptureWildStateIfAny` từ M58
  đã luôn capture vị trí vô điều kiện cho cả 2 loại map (chỉ RIÊNG building mới loại trừ map vẽ tay, vì
  nội dung bí cảnh do người viết đặt sẵn qua `FixedMapLoader`, không phải player xây).

**Verify:** `Run PawnStateSave Tests` (14 case mới, All/ReplaceAll + IO round-trip + JsonUtility thật +
save cũ không chặn load + Capture bỏ qua ô rỗng) + hồi quy BuildPersist/WildMapPawnState/Bed/
AbsentPartyNeeds/Party đều xanh. **Play Mode thật (2026-08-28):** dời 4 pawn debug tới 4 góc map thật →
`SaveManager.Instance.Create()` NGAY TRÊN map (chưa rời) → đọc file JSON thật có đủ `wildTilePawns` +
4 pawnRef → `DestroyImmediate` sạch `WildMapPawnState` (RAM store) → `SaveManager.Instance.Load()` từ đĩa
→ store tự tái tạo đúng 4 vị trí → `bootstrap.Generate()` sinh lại map → **cả 4 pawn về ĐÚNG TUYỆT ĐỐI
toạ độ đã lưu** qua ĐÚNG `WildMapPawnRestorer` (production path, không gọi tắt).

## M61c — vá "AbsentPartyNeedsSystem + BedClaimRegistry" (2026-08-28, cùng ngày, user yêu cầu tiếp)

User gộp 2 tên trong yêu cầu, nhưng SOÁT KỸ trước khi code lộ ra: **`AbsentPartyNeedsSystem` tự nó KHÔNG
giữ state gì cả** — đọc hết file chỉ thấy `_clock` (cache tham chiếu, rebuild ở `OnEnable`) và `_foodIds`
(cache derive từ `DataRegistry<MaterialDefinition>`, cũng rebuild mỗi `OnEnable`). Dữ liệu THẬT nó cần để
chạy đúng sau khi tắt-mở game — party membership (`PawnPartyRegistry`, đã lưu từ M57 qua `PartyIO`),
Hunger/Rest/Mood (`PawnBody`, đã lưu từ trước qua `DiscipleIO`) — **đều đã persist sẵn**. Chỉ CÒN
`BedClaimRegistry` (M60) là sổ THẬT chưa vào SaveSystem — đây mới là việc cần vá, không phải
`AbsentPartyNeedsSystem`. Nói thẳng điều này với user thay vì tự bịa ra chỗ để "vá" cho đủ 2 cái tên.

**`BedClaimRegistry`** vá ĐÚNG pattern `WildMapPawnState`/`WildMapBuildStateStore`: thêm
`BedClaimEntry` (flat struct, mirror `PawnPosition`) + `All`/`ReplaceAll` + `SaveSystem/
BedClaimRegistryIO.cs` (mới) + cắm cả 3 đường Load/Create/WriteFile. **KHÁC 2 sổ theo-ô ở đúng 1 điểm:**
KHÔNG cần `CaptureCurrentSceneIfLoaded` — sổ này ghi TRỰC TIẾP mỗi lần `BedOps.ClaimBed`/`UnclaimBed`
(luôn "sống", không có khái niệm "chụp lúc rời map" như building/vị trí pawn vốn chỉ snapshot LocalMapData
khi rời scene).

⚠ **Bẫy tự dính lúc verify Play Mode — SUÝT báo nhầm thành bug:** verify ban đầu dùng
`SaveManager.Instance.Create(save)` để mô phỏng "đang chơi, bấm Lưu" (đúng cách đã dùng cho M61/M61b) —
kết quả `has=False` sau khi Load lại, dù `WildBuilding.OwnerRef` (Tầng 3) vẫn còn nguyên. Điều tra ra:
KHÔNG phải bug — `Create()` cố ý `Restore()` MỌI sổ về rỗng trước khi ghi (đúng ngữ nghĩa "ván mới"), và
`Create()` trong code thật CHỈ được gọi từ `MainMenuController` lúc "New Game" — **trước khi bất kỳ scene
Wild nào từng load**, nên trong gameplay thật không có claim nào tồn tại ở thời điểm đó để mà mất. Bài học
methodology: verify "bấm Lưu giữa chừng ván đang chơi" phải dùng `SaveManager.Instance.Save()`
(KHÔNG gọi Restore, chỉ `WriteFile` thẳng ActiveSave hiện có), không phải `Create()` — 2 hàm có NGỮ NGHĨA
khác nhau, dùng nhầm hàm sẽ tạo ra "bug giả" giống hệt bug thật.

**Verify:** `Run BedClaimSave Tests` (13 case mới) + hồi quy Bed/PawnStateSave/BuildPersist/
AbsentPartyNeeds/WildMapPawnState/Party đều xanh. **Play Mode thật:** claim giường thật cho pawn thật qua
`BedOps.ClaimBed` (quality 2.6, có bonus nhà ở) → `SaveManager.Instance.Save()` (ĐÚNG mô phỏng "bấm Lưu
giữa chừng", không phải Create) → đọc file JSON thật có `bedClaims` + đúng PawnRef → `DestroyImmediate`
sạch RAM `BedClaimRegistry` → `SaveManager.Instance.Load()` từ đĩa → claim khôi phục ĐÚNG TUYỆT ĐỐI
(quality 2.6, count=1).

## M61d — vá nốt Zone + Designation (2026-08-28, cùng ngày, user yêu cầu tiếp)

Mở rộng NGAY trên kiến trúc M61 sẵn có (`WildMapBuildState`/`WildMapBuildStateStore`) thay vì dựng hệ
song song — Zone/Designation vốn cũng sống trong `LocalMapData` theo Ô WORLD y hệt building, chỉ là bị
cắt scope có chủ đích ở M61 ban đầu.

- **Zone** (`LocalMap/Zoning/Zone.cs`/`ZoneManager.cs`) — `ZoneSnapshot` (flat, tiles là
  `List<ZoneTileEntry>` vì `Zone.Tiles` là `HashSet<Vector2Int>`) + `ZoneManager.ReplaceAll(zones,
  nextIdHint)` (mirror HỆT `PawnPartyRegistry.ReplaceAll` — tự suy `_nextId` từ id lớn nhất "zone:N" nếu
  hint ≤ 0). Restore dựng lại `Zone` object rồi gọi `AddTile(cell, grid)` cho từng ô — vừa nạp tile vào
  zone VỪA re-apply `Cell.ZoneId` lên `GridSystem` (grid tái sinh mới hoàn toàn nên KHÔNG tự nhớ gì).
  `zonesNextId` lưu PER-TILE (mỗi `LocalMapData` có `ZoneManager` RIÊNG, không cần duy nhất toàn cục như
  `party:N`). **Cosmetic gap CHẤP NHẬN ĐƯỢC, nói thẳng:** `_nameCounters` (bộ đếm tên đề xuất "Vùng Trồng
  N") KHÔNG lưu — chỉ ảnh hưởng SỐ trong tên GỢI Ý cho zone mới tạo SAU restore, không đụng ZoneId/dữ liệu.
- **Designation** (`World/Wild/Designations/Designation.cs`/`DesignationManager.cs`) — `DesignationSnapshot`
  (vị trí + workGiverId + `targetFeatureType` string) + `DesignationManager.ReplaceAll`. **Điểm khó nhất:**
  `Designation.TargetFeature` là REFERENCE tới 1 `MapFeature` sống trong `LocalMapData.Features` — không
  lưu được object thẳng (JsonUtility + vòng đời khác nhau), nên chỉ lưu `FeatureType` (string) rồi
  Restore TỰ TÌM LẠI feature khớp CẢ vị trí LẪN loại trong `Features` vừa sinh lại. Đúng vì Feature sinh
  deterministic từ seed nên VẪN CÒN ĐÓ sau reload — TRỪ khi đã bị hái/xoá khỏi danh sách trước lúc rời map
  (ca đó fallback về `null` AN TOÀN, đúng ĐÚNG hành vi gốc `TargetFeature==null` — không phải lỗi).
  **KHÔNG lưu `Claimed`** — cùng lý do `BuildPiece`/`WildBuilding` không lưu `Claimed`/`ReservedForHaul`:
  về `false` là ĐÚNG khi map vừa nạp lại (chưa pawn nào kịp nhận việc), pawn tự nhận lại qua
  `WorkAssigner` bình thường.
- Cả hai cắm vào ĐÚNG `WildMapBuildStateStore.Capture`/`Restore` đã có — KHÔNG thêm bước pipeline mới,
  KHÔNG sửa `WildSceneBootstrap`/`SaveManager` (đã đủ từ M61/M61b/M61c).

**Verify:** mở rộng `Run BuildPersist Tests` lên **44 case** (13 case mới: Zone full round-trip + grid
`Cell.ZoneId` re-apply + `NextId` counter, Designation với feature TRÙNG VỊ TRÍ KHÁC LOẠI resolve đúng
target cụ thể, Designation fallback null khi không có target, Designation fallback null khi feature đã bị
hái mất trước lúc rời map, JsonUtility thật cho cả Zone+Designation lồng trong `SaveData`). Hồi quy
BedClaimSave/PawnStateSave/Bed/AbsentPartyNeeds/Party/FixedMap/BuildPiece/JobGiverConstruction/
WildBuilding/Production đều xanh (218+ case).

**Play Mode thật (2026-08-28):** vẽ 1 Zone Farming 2×2 thật (`ZoneManager.PaintRectZone`, API player
click-kéo thật dùng) + 1 Designation thật trên map seed=1337 tile(5,7) → **`Save()` giữa chừng** (mô
phỏng đúng "bấm Lưu mid-game", bài học từ M61c) → đọc file JSON thật có `zones`/`designations` → xoá sạch
RAM `WildMapBuildStateStore` → `Load()` từ đĩa → sinh lại map → Zone khôi phục ĐÚNG TUYỆT ĐỐI (tên/tiles/
crop/`Cell.ZoneId` trên grid mới/`NextId`), Designation khôi phục đúng workGiverId (target feature rơi về
`null` vì feature test được CHÈN TAY không nằm trong worldgen thật — nhánh "resolve thành công" đã verify
qua unit test với feature THẬT tái tạo lại, không phải giả định).

## M61e — vá nốt BeastPopulationStore (2026-08-28, cùng ngày — user: "có bao nhiêu chức năng cần thì vá save hết đi")

Yêu cầu này đòi hỏi SOÁT TOÀN DỰ ÁN trước khi code, không chỉ vá tiếp theo trí nhớ. Cách soát: grep các
cụm tự-thú-nhận "RAM-only"/"CHỖ CẮM để SaveSystem"/"sẵn sàng cho SaveSystem" xuyên `Assets/Scripts` — đây
là cách đáng tin cậy nhất vì dự án có thói quen ghi rõ nợ persistence ngay trong doc comment (đã thấy nhiều
lần các đợt trước). Kết quả: **đúng 1 chỗ còn lại** — `World.Wild.Beasts.BeastPopulationStore` (M12/M38),
tự ghi trong doc class *"Records đều [Serializable] → là CHỖ CẮM để SaveSystem serialize xuống đĩa về sau"*
— nợ cũ hơn CẢ Tầng 1 (có từ M12, trước M58 rất lâu). Mọi registry DontDestroyOnLoad khác (`PawnRoster`,
`PawnPartyRegistry`, `CompanionRegistry`, `WorldEventStore`, `ActorRegistry`, `WorldChronicle`,
`MemoryRegistry`) đã có IO tương ứng từ trước đợt này.

- **Kiến trúc** — mirror ĐÚNG pattern `WildMapPawnState`: `BeastActor` vốn ĐÃ `[Serializable]` với field
  public thuần (không reference cần resolve lại như `Designation.TargetFeature`) nên chỉ cần bọc phẳng
  `BeastTileState{tileKey, actors}` + `All`/`ReplaceAll` trên `BeastPopulationStore` +
  `SaveSystem/BeastPopulationStoreIO.cs` + cắm 3 đường Load/Create/WriteFile.
- **`CaptureCurrentSceneIfLoaded` VẪN CẦN** dù `BeastSpawner._store.Save(_tileKey, _actors)` giữ THẲNG
  reference (vị trí luôn live, không cần flush) — nhưng **HP thì KHÔNG**: `BeastActor.hp` chỉ là gương
  của `PawnBody.CurrentHP` (M38), đồng bộ MỘT LẦN lúc `BeastSpawner.SaveBack()` chạy (rời map/regenerate/
  quái chết). Bấm Lưu giữa trận đánh mà không có bước flush này thì máu vừa trừ sẽ KHÔNG kịp xuống field
  lưu — thêm `BeastSpawner.FlushToStore()` (public wrapper của `SaveBack` cũ) để
  `BeastPopulationStore.CaptureCurrentSceneIfLoaded()` gọi được từ `SaveManager.WriteFile`.
- **KHÔNG cần sửa `BeastSpawner.HandleGenerated`** — hàm đó ĐÃ BIẾT đọc từ `_store.Get(_tileKey)` khi map
  sinh (thiết kế gốc M12: "rời localmap không mất quái" luôn cần đọc store, RAM hay đĩa không quan trọng
  với nó) — chỉ cần STORE có đúng dữ liệu SAU KHI game khởi động lại là đủ, không đụng pipeline sinh map.

**Verify:** `Run BeastSave Tests` (14 case mới, gồm field "hiếm" — `deathDay`/`questGuaranteedDropId`/
`packId` âm — sống sót qua JsonUtility thật) + hồi quy BuildPersist/BedClaimSave/PawnStateSave/Bed/
AbsentPartyNeeds/Party/FixedMap đều xanh. **Play Mode thật:** map thật (seed=1337, tile 5,7) không tự
spawn quái ở vùng test nên tiêm 1 `BeastActor` thật vào `_actors` LIVE của `BeastSpawner` qua reflection
(list đã CHIA SẺ reference với store từ `Awake()`, xác nhận `store.Has()` đúng NGAY không cần gọi gì
thêm) → `Create()` mid-session (wipe rồi tự phục hồi qua `CaptureCurrentSceneIfLoaded`, giống M61/M61b,
KHÁC M61c) → đọc JSON thật có quái → `DestroyImmediate` sạch RAM store → `Load()` từ đĩa → sinh lại map
qua ĐÚNG `bootstrap.Generate()` → `BeastSpawner.Actors` có lại đúng con quái, đúng vị trí (105,105), đúng
HP (42) — qua chính `HandleGenerated`, không gọi tắt.

## Nợ còn lại (nói thẳng — KHÔNG phải "mọi tính năng đều đã vào save")

- Feature tự nhiên đã hái/đào (TRẠNG THÁI Depleted/RegrowAtDay của `MapFeature`) và ground-item-equipment:
  xem mục Scope M61 gốc — cắt có chủ đích, chưa ai yêu cầu vá.
- ~~Zone/Designation~~ — **đã vá ở M61d.** ~~Tầng 1 (vị trí pawn)~~ — **đã vá ở M61b.**
  ~~AbsentPartyNeedsSystem/BedClaimRegistry~~ — **đã vá ở M61c** (AbsentPartyNeedsSystem hoá ra không cần
  vá — không giữ state). ~~BeastPopulationStore~~ — **đã vá ở M61e.**
- **`WildQuestBoard` (Wild quest board) CỐ Ý KHÔNG vá, đã báo user riêng — khác lớp vấn đề:** đây KHÔNG
  phải "registry bền quên chưa lưu" như 5 mục trên — nó **sống theo SCENE** (design gốc M40: mỗi lần map
  Wild sinh lại là board TỰ SEED LẠI quest từ đầu qua `Awake()→SeedDemoQuests()`, kể cả trong CÙNG phiên
  chơi, KHÔNG dùng DontDestroyOnLoad). Muốn quest tiến trình (đã nhận/đã hoàn thành) sống qua reload là
  một QUYẾT ĐỊNH KIẾN TRÚC khác (đổi vòng đời board từ per-scene sang per-save), không phải thêm 1 IO file
  theo pattern đã có — cần user xác nhận trước khi động vào.
- **City sim persistence** (nhắc tới trong CLAUDE.md §7 "City lưu chỉ seed+tier+delta player + CityState")
  — grep xác nhận KHÔNG có `CityState`/save wiring nào tồn tại trong `SaveSystem/` — mô tả đó có vẻ là
  THIẾT KẾ DỰ ĐỊNH chưa từng cài đặt, không phải nợ có sẵn đã biết rõ hình dạng. Việc này lớn hơn hẳn
  "thêm 1 IO file" (cần khảo sát lại toàn bộ City sim hiện tại trước) — CỐ Ý không tự ý làm, cần hỏi user.
- Chưa có UI/thông báo nào cho player biết các sổ này tồn tại — thuần là hạ tầng câm, đúng bản chất mọi
  lớp persist khác trong dự án (không cần thông báo, chỉ cần ĐÚNG).
