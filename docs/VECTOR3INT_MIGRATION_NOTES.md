# Vector2Int → Vector3Int migration — nhật ký (2026-09-12)

> Bối cảnh đầy đủ: [`CLAUDE.md` TL;DR](CLAUDE.md) ⚠ QUAN TRỌNG + memory `project_voxel_z_future_proofing`.
> Mục tiêu: nâng hệ cell 2D lên voxel 2 tầng (z=0 ground / z=1 non-ground) để phòng khi dự án lên 3D thật
> (~2028-2030), không phá API modder. `Vector2Int` (built-in Unity) là type tọa độ dùng khắp ~251 file —
> không có struct tự viết để chêm field z, nên đường đi là đổi hẳn sang `Vector3Int` (Unity có sẵn, z mặc
> định 0 = ground).

**An toàn:** repo đã `git init` + baseline commit `b8010d4` TRƯỚC khi đụng file nào. Save/JSON đã an toàn
sẵn từ trước (mọi snapshot dùng `int x,y` phẳng, không serialize `Vector2Int` trực tiếp) — xem
`WildMapBuildState.cs:158`.

## Đã đổi xong, compile sạch tới bước này

1. **`LocalMap/Grid/Cell.cs`** — `Position` property: `Vector2Int` → `Vector3Int(X,Y,0)`.
2. **`LocalMap/Grid/GridSystem.cs`** — toàn bộ API (`At`, `InBounds`, `GetCopy`, `SetTileType`,
   `SetPassable`, `SetRockType`, `SetLingQiModifier`, `PlaceBuilding`, `RemoveBuilding`, `AssignZone`,
   `ClearZone`, `GetNeighbors`, event `CellChanged`) — đây là **gốc thật sự** của cả đợt migration.
3. **`LocalMap/Zoning/Zone.cs`** — `_tiles`/`Tiles`/`AddTile`/`RemoveTile`/`Contains` toàn bộ sang
   `Vector3Int` (Zone tile về bản chất LÀ vị trí grid, hợp lý đổi trọn thay vì convert biên).
4. **`LocalMap/Zoning/ZoneManager.cs`** — toàn bộ (`PaintRectZone`/`EraseRect`/`RemoveZone`/`ZoneAt`).
5. **`World/Wild/Work/WorkPositioning.cs`** — toàn bộ (`ResolveApproachTile`/`ResolveAdjacentTile`/
   `ResolveAdjacentToFootprint`/`ResolveApproachToFootprint`/`ParseFixedDirection`) — file tự chứa, không
   phụ thuộc type ngoài nào khác.
6. **`World/Wild/Zones/WildZoneOps.cs`** — **CHỈ convert biên** (`new Vector3Int(cell.x, cell.y, 0)` tại
   điểm gọi `Grid.GetCopy`), **KHÔNG đổi chữ ký public** (`IsPlantable`/`TryPlantOne` vẫn nhận
   `Vector2Int cell`) — vì nó so sánh trực tiếp với `MapFeature.Position` (vẫn `Vector2Int`, xem dưới).

## CẬP NHẬT — thử "skip hệ lớn, làm hệ nhỏ" thì phát hiện KHÔNG CÒN hệ nhỏ độc lập

Đã thử đổi thêm `Combat/Encounter/CombatGrid.cs` (37 file trong `Combat/*`, ban đầu tưởng tách biệt hoàn
toàn — không `using Nizhangjie.LocalMap` file nào). Recompile lộ ra: **267 lỗi trải khắp ~80 file** — không
chỉ Combat, mà kéo theo TOÀN BỘ `LocalMap/Pathfinding` (`PathGrid`, `LocalPathfinder`), gần hết
`World/Wild` (mọi `JobGiver_*`, các Generation Pass, UI: `WildSceneInteraction`, `WildTilemapRenderer`,
`WildPlayerToken`...), và `World/City/Generation` (`CityGenerator`, `CityFurnisher`, `CityYardDecorator`).

**Kết luận quan trọng:** `GridSystem`/`Cell` không phải 1 hệ trong nhiều hệ — nó là **mạch máu chung** mà
gần như toàn bộ gameplay (di chuyển, AI, xây dựng, combat, world-gen) đi qua. Không có "hệ nhỏ độc lập"
nào còn sót để nhặt tiếp — đụng gốc = đụng gần hết game. Yêu cầu ban đầu "skip hệ lớn, làm hệ khác nếu
còn" hoá ra không áp dụng được nữa vì thực tế chỉ còn 1 khối duy nhất, không phải nhiều khối rời.

**Đã dừng lại hoàn toàn, không sửa thêm gì.** Số liệu 267 lỗi/~80 file là cơ sở thật để quyết định: đây là
việc cho 1 phiên riêng, có giám sát, đi từng lớp một (Combat → Pathfinding → JobGivers → City-gen) với
Play Mode test giữa chừng — không phải việc "quất cho hết" trong 1 đêm không ai theo dõi.

## CẬP NHẬT 2 — Pathfinding lõi ĐÃ XONG, verify sạch qua MCP (2026-09-12)

User yêu cầu cụ thể: chỉ làm nhóm "Pathfinding lõi", không đụng thêm gì khác. Đã đổi trọn 7 file
(không phải 4 như liệt kê ban đầu — `RegionReachability.cs`, `IPathfinder.cs`, `LocalPathfinderAdapter.cs`
cũng nằm trong cùng 1 mạch, tách riêng sẽ không compile được):

- `LocalMap/Pathfinding/IPathfinder.cs`
- `LocalMap/Pathfinding/LocalPathfinderAdapter.cs`
- `LocalMap/Pathfinding/LocalPathfinder.cs`
- `LocalMap/Pathfinding/PathGrid.cs`
- `LocalMap/Pathfinding/PathRegionGrid.cs`
- `LocalMap/Pathfinding/RegionReachability.cs`
- `LocalMap/Pathfinding/RegionPathfinder.cs`

**Verify qua Unity MCP (`read_console`, không đoán):** sau khi đổi, compile lại — **0 lỗi trong
`LocalMap/Pathfinding/*`**. Tổng lỗi toàn project tăng 267→312 (KHÔNG phải regression — đây là lớp
cascade MỚI lộ ra: các hàm giờ trả `List<Vector3Int>` (`FindPath`, `FindReachableSet`...) không còn khớp
kiểu `List<Vector2Int>` mà caller (`WorkAssigner`, `CityNpcSystem`, `IdleActivityPipeline`, các
`JobGiver_*`...) đang giữ — đúng như dự đoán "sửa gốc lộ thêm 1 tầng downstream").

**Đã dừng lại đúng như yêu cầu — CHƯA đụng gì ngoài 7 file trên.** Toàn project vẫn CHƯA compile được
(312 lỗi ở Combat/JobGivers/City-gen/UI) — nhóm Pathfinding chỉ là 1 bước trong chuỗi, không phải đích
cuối. Việc tiếp theo (khi được yêu cầu): cụm Combat (~25 file) hoặc trực tiếp các file downstream vừa lộ
ra (`WorkAssigner`, `CityNpcSystem`, `IdleActivityPipeline`, `JobGiver_*`) — cách làm giống hệt: đổi
`Vector2Int`→`Vector3Int` trọn file nếu file đó thuần vị trí, hoặc convert biên nếu file đó chạm type
ngoài phạm vi (vd `MapFeature.Position`).

## CẬP NHẬT 3 — `Combat/*` HOÀN TẤT, compile sạch 100% (2026-09-12)

User giao "trả bài cho Combat" — đã đổi hết toàn bộ 27 file trong `Assets/Scripts/Combat/*` (22 file ban
đầu + 5 file lộ thêm giữa chừng do cascade nội bộ Combat: `UnitVisual`, `TickMover`,
`Events/CombatGameplayEvents` (KnockbackEvent/KnockbackCollisionEvent), `CombatCoreDriver`,
`Commands/{PauseCommandQueue,SkillCommand,ComboChain}`, `Drivers/{CombatSetup,RtwpCombatDriver}`,
`AI/CombatBehaviorSystem`, `Skills/{SkillCastController,SkillExecutor}`, `Units/UnitFactory`,
`Visual/{UnitAttackIndicator,UnitCastBar}`, `RTwP/CombatRTwPController`) + 1 field ngoài Combat nhưng
tightly-coupled (`Core/Events/SampleEvents.cs` — `SkillCastEvent.TargetTile`, chỉ 1 nơi dùng, an toàn).

**Verify qua Unity MCP:** 0 lỗi compile trong `Assets/Scripts/Combat/*`. 227 lỗi còn lại TOÀN BỘ nằm
ngoài phạm vi này — `World/Wild/*`, `World/City/*`, `World/Hamlet/*`, và `UI/Combat/FloatingCombatTextUI.cs`
(khác thư mục `Combat/` gốc — thuộc UI, chưa đụng).

**Bẫy đã gặp giữa chừng — CẨN THẬN LẦN SAU:** dùng PowerShell `-replace`/`Set-Content -Encoding utf8` để
sửa `CombatCoreDriver.cs` (fix constructor 2-tham-số) làm HỎNG ENCODING toàn bộ comment/string tiếng Việt
trong file (mojibake — UTF-8 bị đọc nhầm thành codepage khác rồi ghi lại). Đã phục hồi bằng Write tool
(ghi lại nguyên văn UTF-8 đúng). **TỪ NAY: chỉ dùng Edit tool cho mọi sửa đổi trong file có comment tiếng
Việt — không dùng PowerShell `-replace`/`Set-Content` cho file này**, đúng bẫy đã ghi trong memory
`feedback_powershell_utf8`.

**Việc tiếp theo (khi được giao):** `World/Wild/*` là cụm lớn nhất còn lại (~150 lỗi, chủ yếu JobGivers +
Generation Passes + UI), sau đó `World/City/*` (~15 lỗi). Cách làm giống hệt: đổi trọn `Vector2Int`→
`Vector3Int` cho file thuần vị trí, convert biên cho file chạm type ngoài phạm vi (như đã làm với
`WildZoneOps`/`MapFeature.Position`).

## Nhóm lỗi ban đầu (10 lỗi, trước khi thử Combat) — vẫn còn nguyên trong 267 lỗi trên

Compile hiện tại còn đúng **10 lỗi CS1503** ở 4 file:

- `World/Wild/Work/WorkAssigner.cs` (dòng 114/117/118)
- `LocalMap/Pathfinding/RegionPathfinder.cs` (dòng 96/159)
- `LocalMap/Pathfinding/PathRegionGrid.cs` (dòng 93/108)
- `World/Wild/WildPartyTestSpawner.cs` (dòng 412)

**Lý do dừng:** `RegionPathfinder`/`PathRegionGrid` là **hệ pathfinding lõi** — đa luồng
(`PrepareQuery`/`ExecutePreparedQuery`, `Version` chống race, xem comment trong chính file đó), dùng cho
CẢ City lẫn Wild. Đổi API của nó (`FindPath`, `PrepareQuery`, `PreparedQuery.From/To`, `RegionAt`,
`Reachability.CanReach`...) chắc chắn kéo theo hàng chục file khác (`CityNpcSystem`, `LocalPathfinder`,
`RegionReachability`, `CellAStar`...) — quy mô nhảy vọt so với các nhóm nhỏ ở trên, và có rủi ro thật đụng
vào bất biến thread-safety đã cẩn thận cài (5 vòng tối ưu hồi 2026-08-08, xem `project_city_commute_perf`
trong memory). Đây đúng là ngưỡng "lớn quá" — không nên sweep tiếp lúc không ai giám sát/Play Mode test.

**Tương tự:** `WildZoneOps.IsPlantable` chạm tới `MapFeature.Position` (`LocalMap/Features/MapFeature.cs`,
vẫn `Vector2Int`) — hệ Feature (crop/scatter/scan) cũng là 1 subtree rộng chưa đụng tới.

## Việc cần làm tiếp (khi có người giám sát)

1. Sửa 4 file lỗi ở trên — cách sửa TƯƠNG TỰ nhóm 1 (đổi kiểu tham số cục bộ hoặc convert biên tại điểm
   gọi), nhưng làm CẨN THẬN qua từng bước vì đụng pathfinding đa luồng.
2. Sau khi pathfinding xanh, chắc chắn còn cascade tiếp vào `CityNpcSystem`/AI/Combat — mỗi vòng recompile
   qua `mcp__UnityMCP__refresh_unity` + `read_console` (types=["error"]) để biết chính xác bước tiếp.
3. **Play Mode test bắt buộc trước khi coi là xong** (theo `feedback_done_means_playable` — compile sạch
   KHÔNG phải xong): thử di chuyển pawn, vẽ Zone, đặt building ở cả Wild lẫn City.
4. `MapFeature.Position` (Vector2Int) là subtree riêng, rộng — nên làm thành 1 phiên riêng, không gộp
   chung với đợt pathfinding.

## Cách tiếp tục / hoàn tác

```bash
# Xem đúng những gì đã đổi so với baseline
git diff b8010d4 -- Nizhangjie/Assets/Scripts

# Hoàn tác TOÀN BỘ đợt migration này về baseline sạch
git checkout b8010d4 -- Nizhangjie/Assets/Scripts

# Hoàn tác 1 file cụ thể
git checkout b8010d4 -- Nizhangjie/Assets/Scripts/LocalMap/Grid/GridSystem.cs
```

Working tree hiện **chưa commit** (cố ý — để bạn xem `git diff` trước khi quyết định commit hay bỏ).

## CẬP NHẬT 4 — `World/Wild/*` HOÀN TẤT, 0 lỗi (2026-09-12, phiên tối)

User giao tiếp "World/Wild là đích, quất cho hết trong đêm không cần giám sát" — đã dọn sạch toàn bộ
~280 lỗi ban đầu (`WildSceneInteraction`/`WildTilemapRenderer` chiếm nhiều nhất) bằng đúng chiến lược đã
ghi ở CẬP NHẬT 2/3: đổi trọn `Vector2Int`→`Vector3Int` cho file thuần vị trí trong `World/Wild/*`, convert
biên tại điểm gọi cho những chỗ chạm type ngoài phạm vi (City/Hamlet/CaptureOps/ITileWorldMapper).

**Bài học lớn nhất phiên này — KHÔNG có "hệ nhỏ độc lập" nào còn sót trong `World/Wild`, y hệt kết luận
CẬP NHẬT 2 cho `GridSystem`:** sweep bulk-regex trên các file "đầu mối" (`JobGiver_*`, `WildCombatOps`,
`BeastBehaviour`...) tự nó KHÔNG đủ — chúng vẫn Vector2Int-nội-bộ NHƯNG lỗi thật nằm ở các struct/class
"mạch máu" chưa từng lọt vào danh sách lỗi ban đầu vì tự chúng biên dịch được (nội bộ nhất quán
Vector2Int), chỉ vỡ ra SAU KHI cascade tới. Danh sách các "mạch máu ẩn" phải đổi trọn thêm (ngoài các file
đã liệt ở đầu task):

- **`World/Wild/Ai/PawnAiContext.cs`** — struct gói tham số CHO CẢ 17 JobGiver (`Cell`, `FeatureAt`,
  `NearbyBeasts`). Không đổi trọn thì MỌI JobGiver lỗi hàng loạt dù bản thân JobGiver không hề chứa chữ
  "Vector2Int" nào.
- **`World/Wild/Combat/WildCombatant.cs`** (`IWildCombatant.Cell`) + **`World/Wild/Companions/
  {CompanionActor,CompanionCombatant,CompanionOps}.cs`** — cùng 1 interface `Cell`, đổi 1 bên không đổi
  bên kia là lỗi "does not implement interface member" (CS0738), không phải lỗi Vector2Int/Vector3Int
  thường thấy.
- **`World/Wild/Beasts/{BeastActor,BeastAI,BeastGoals}.cs`**, **`World/Wild/Building/{WildProducer,
  WildWorkstation,Functions/BuildingActivationArgs}.cs`**, **`World/Wild/Combat/{BeastCombatMemory,
  WildCombatFx}.cs`**, **`World/Wild/Generation/{LocalMapData,RoofArea}.cs`**, **`World/Wild/Persistence/
  WildMapBuildState.cs`**, **`World/Wild/UI/WildRoofLens.cs`** — tất cả tự biên dịch sạch một mình (0 lỗi
  ban đầu) nhưng là dữ liệu NỀN (`LocalMapData` đặc biệt quan trọng — Roofs/GroundTint/Exits/EntryPoint)
  mà UI/AI/Combat/Persistence phía trên đọc — phải đổi trọn theo, không convert biên nổi vì chúng LÀ nguồn.
- **`Core/Pawns/Capture/CaptureOps.cs`** — CỐ Ý giữ `Vector2Int` (dùng chung với capture-tool project,
  ngoài phạm vi Wild-only) — 3 call site chạm nó (`WildPlayerToken.FaceToward/TryStep/Update`,
  `CorePawnInteractions.ThrowCaptureForward`) convert biên tại chỗ gọi, KHÔNG đổi `CaptureOps`.

**Ranh giới `ITileWorldMapper` (`World/UI/ITileWorldMapper.cs`) giữ nguyên Vector2Int đúng như task yêu
cầu** — `WildTilemapRenderer.{RenderCell,HighlightCell,HighlightRect}` vẫn nhận `Vector2Int`, nhưng TOÀN
BỘ logic bên trong (`PaintGroundCell`, `FindFeatureAt`, `FindBuildPieceAt`, `FindWildBuildingAt`,
`data.Designations.GetAt`...) đã là `Vector3Int` từ đợt trước — điểm chuyển đổi DUY NHẤT là 1 dòng
`var cell = new Vector3Int(pos.x, pos.y, 0);` ngay đầu `RenderCell`, dùng `cell` cho mọi lookup nội bộ
thay vì `pos` (bug thật: bản trước đó lẫn lộn `pos`/`cell` giữa các dòng, gây đúng loại lỗi CS1503 hàng
loạt mà việc sweep không tự sinh ra — phải sửa tay).

**Bài học "không có implicit conversion Vector2Int→Vector3Int" áp dụng NGƯỢC ở `RiverCarverPass.cs`/
`RoadStubPass.cs`:** 2 file này cast `Vector3Int` sang `Vector2` (KHÔNG PHẢI `Vector2Int`) để tính toán
Bezier/khoảng cách bằng số thực — Unity CÓ explicit operator `Vector2(Vector3Int)` nhưng KHÔNG có
implicit, và bản thân `(Vector2)a` (a: `Vector3Int`) KHÔNG compile thẳng — phải ép 2 tầng
`(Vector2)(Vector3)a`. Đây là biến thể MỚI của bẫy "không implicit conversion" mà task đã cảnh báo,
chưa từng gặp ở các file trước (toàn convert Vector2Int↔Vector3Int, chưa ai đụng Vector2 thường).

**Collateral tối thiểu ngoài `World/Wild/*` (đúng luật "chỉ sửa điểm chạm, không convert trọn file"):**

| File (ngoài scope) | Chỗ sửa |
|---|---|
| `Sim/Npc/NpcSpot.cs`, `NpcGenerator.cs` | 4 chỗ đọc `WildBuilding.Origin/Footprint` (nay Vector3Int) — convert biên khi tra `Dictionary<Vector2Int,...>`/`HashSet<Vector2Int>` nội bộ City. |
| `UI/Combat/FloatingCombatTextUI.cs` | 1 chỗ gọi `GridVisual.FootprintCenterWorld` (Combat, đã Vector3Int) — đổi `Vector2Int.one` → `new Vector3Int(1,1,0)`. |
| `World/Hamlet/HamletSpawner.cs`, `Interaction/HamletResidentProvider.cs` | 4 chỗ — `GridSystem.InBounds/At` giờ Vector3Int, `WildBuilding` ctor giờ Vector3Int; hàm nội bộ `FindPassableCellsNear` GIỮ NGUYÊN Vector2Int (theo đúng chỉ dẫn task), chỉ convert biên. |
| `World/City/Generation/{SettlementStampPass,CityGenerator,CityBuildPieceStamper,CityLocalMapAdapter,CityYardDecorator,CityFurnisher}.cs` | Mọi chỗ gọi `GridSystem.SetTileType/SetPassable/PlaceBuilding/RemoveBuilding`, `LocalMapData.AddRoof/SetGroundTint/Exits/EntryPoint`, `WildBuilding` ctor — convert biên (thêm helper `private static IEnumerable<Vector3Int> ToV3(...)` cục bộ trong `CityGenerator` cho các đoạn lặp dài). |
| `World/City/Interaction/{CityLotProvider,CityBuildingProvider}.cs` | `Collect(..., Vector3Int cell, ...)` (interface đã đổi) tra `Dictionary<Vector2Int,...>` nội bộ — convert biên. |
| `World/City/Npc/CityNpcSystem.cs` | Cụm lớn nhất: `IsWalkable`, `DrainPathQueue` (RegionPathfinder API), `SpeedFactorAt` delegate, và viết lại hẳn `BeginIndoorWalk` (tính toán bằng `Vector3Int` khớp `WorkPositioning`/`RegionPathfinder`, chỉ convert về `List<Vector2Int>` đúng 1 lần khi gán `npc.IndoorPath` — field đó thuộc `NpcRecord`, ngoài scope). |
| `Tests/Editor/M11CityInteractionTest.cs` | `FakeFurnitureProvider` (implement `ITileInteractableProvider`) đổi `Vector2Int`→`Vector3Int` cho khớp interface mới — bulk-convert nốt phần còn lại của file (test City, không phải Wild) ở CẬP NHẬT 5 bên dưới. |

**Kết quả verify qua Unity MCP (`refresh_unity` force + `read_console` types=["error"]):** `filter_text:
"World\\Wild"` trả về **0 dòng** — `Assets/Scripts/World/Wild/*` compile sạch tuyệt đối.

## CẬP NHẬT 5 — cascade sang `Assembly-CSharp-Editor` (Tests/Editor), CỐ Ý DỪNG ở đây

Sau khi `World/Wild/*` sạch, `refresh_unity` lộ ra **500+ lỗi mới (bị cap ở 500), toàn bộ nằm trong
`Assets/Scripts/Tests/Editor/*`** — ĐÂY LÀ ASSEMBLY KHÁC (`Assembly-CSharp-Editor`, Unity tự tách theo
folder "Editor", không cần asmdef tường minh) phụ thuộc NGƯỢC vào `Assembly-CSharp` (runtime, chứa
`World/Wild`) chứ không ngược lại — nghĩa là **`World/Wild` biên dịch sạch và DÙNG ĐƯỢC ngay cả khi
Tests/Editor còn lỗi**, đúng model biên dịch mặc định của Unity (không phải giả định, đã verify: dừng sửa
Tests giữa chừng rồi `refresh_unity` lại — `filter_text: "World\\Wild"` vẫn 0 dòng dù Tests vẫn đỏ).

Đã tiện tay bulk-convert 34 file test (`AmbientTest`, `BuildPieceTest`, `JobGiver*Test` ×10,
`WildCombatTest`, `WorkAssignerTest`, `M13ZoneTest`, `M61BuildPersistTest`, `M60BedTest`,
`M58WildMapPawnStateTest`, `StorageFunctionTest`, `WildBuildingTest`, `CompanionTest`, `MCombatSkillTest`,
`MCombatAITest`, `M3CombatTest`, `M31LootTest`, `M31PassiveTest`, `M33FixedMapTest`, `NeedsTest`,
`FoodSpoilageTest`, `CorpseTest`, `IdleActivityPipelineTest`, `JobDriverTest`, `CityYardDecoratorTest`,
`M11CityInteractionTest`, `M11InteriorTest`, `M63CityTerrainTest`) qua `Vector2Int`→`Vector3Int` +
`fix_vector3int_args.js`, giảm được một mảng lớn — nhưng recompile lại LỘ THÊM một lớp test KHÁC chưa từng
đụng tới trong TOÀN BỘ đợt migration này (không riêng phiên này): `RegionPathfinderTest.cs`,
`MCombatCoreTest.cs`, `ProductionTest.cs`, `MNpcTest.cs`, `MNpcBrainTest.cs`, `PawnAiBrainTest.cs`,
`M_GameplayTest.cs`, `ResourcePoolTest.cs` — test cho `LocalMap/Pathfinding`, `Combat/*` core, `Sim/Npc`,
những cụm mà CẬP NHẬT 2/3 từng báo "đã xong, verify sạch" (nhưng chỉ verify sạch ở `Assembly-CSharp`,
CHƯA BAO GIỜ verify `Assembly-CSharp-Editor` — lỗ hổng quy trình xuyên suốt cả 3 phiên trước, không phải
lỗi mới phát sinh phiên này).

**CỐ Ý DỪNG LẠI ở đây, không đuổi tiếp** — đúng ranh giới nhiệm vụ được giao (`World/Wild/*` sạch, xác nhận
bằng `filter_text`) và đúng tinh thần "lớn quá thì dừng, để phiên có giám sát" của CẬP NHẬT 2. Việc sửa
toàn bộ `Tests/Editor` sạch 100% là một khối lượng công việc RIÊNG, xuyên suốt nhiều hệ không liên quan gì
đến Wild (`Combat` core, `Pathfinding`, `Sim/Npc`, `City`) — dồn vào task "World/Wild" là lạc phạm vi.
**Việc cần làm tiếp (phiên sau, có giám sát):** lặp lại đúng quy trình `refresh_unity`→`read_console`→sửa
theo cụm cho `Tests/Editor/*` tới khi `filter_text: "Tests\\Editor"` cũng về 0 — ước lượng còn ít nhất
8 file/vài trăm lỗi nữa dựa trên đợt cap 500 vừa thấy (con số THẬT có thể cao hơn, chưa đếm hết vì bị cap).

**Dọn dẹp:** xoá 3 file rác từ phiên trước còn sót ở root project (`bulk_replace.js`, `core_types.txt`,
`wild_files.txt`) — giữ nguyên `fix_vector3int_args.js` (dùng lại được, đã ghi rõ trong task).
