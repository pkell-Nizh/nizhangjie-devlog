# M10 — Procedural World Generation Pipeline

> ⚠️ **LỖI THỜI (2026-07-03, refactor CellIndices):** doc này mô tả `WorldTile` (class `[,]`) —
> đã thay bằng **struct `Cell`** trong mảng phẳng `WorldMap.Cells` (`map.At(x,y)` ref, KHÔNG
> `map.Get(x,y)` trả class). `BiomePass` KHÔNG còn hardcode if/else — data-driven qua window
> elev/temp/moist của `BiomeDefinition` (mod thêm biome mới không cần sửa code). Đã thêm
> `RegionPass` (Phase 4 — lãnh địa), `ApplyHydroModifiersPass` (Phase 3 — sông ảnh hưởng
> Danger/Resource), `Event Pool` (Phase 5 — `World/Events/EventPoolResolver`). Nguồn sự thật
> hiện tại: `CLAUDE.md` §5 + `nizhangjie-cellindices-plan.md` + đọc thẳng code trong
> `World/Cell.cs`, `World/WorldMap.cs`, `World/Generation/Passes/`. Đừng copy code mẫu bên
> dưới — dùng để hiểu Ý ĐỊNH pipeline (thứ tự pass, lý do thiết kế), không phải API hiện tại.

**Namespace:** `Nizhangjie.World.*` (tầng World, mới) + mở rộng `Nizhangjie.LocalMap.*` (tầng Local, đã có)
**Mục tiêu:** Sinh ra một thế giới tu tiên kiểu RimWorld — bản đồ ô vuông khổng lồ, chân thực, có sông/núi/rừng/biển, biome đa dạng, thành phố & tông môn NPC có sẵn, ải/dungeon/linh mạch, và **mỗi ô world có thể "bước vào" để sinh ra một LocalMap chi tiết 100×100 hoặc 200×200**.

Tài liệu này là **kế hoạch** (không phải code). Style theo M1–M9 README. Bạn tự code theo thứ tự ở mục 12.

---

## 0. Triết lý & ràng buộc kiến trúc

Pipeline **bắt buộc** tuân theo triết lý có sẵn của dự án (xem `PROJECT_SUMMARY.md`):

- **JSON-first** — mọi tham số gen (biome, noise, structure, POI) define trong JSON, không hard-code, không ScriptableObject. Load qua `JsonDataLoader` + `DataRegistry<T>`.
- **Plain C#, testable** — generator là plain C# class, **không** MonoBehaviour. Không phụ thuộc Unity lifecycle → chạy được trong Editor test (như `Tests/Editor/M*Test.cs`).
- **No singletons** — generator nhận dependency qua constructor; kết quả publish qua EventBus nếu cần.
- **Deterministic** — cùng một `seed` → cùng một thế giới. Đây là ràng buộc cứng (xem mục 3).
- **Tái dùng hạ tầng có sẵn** — `GridSystem`, `Tile`, `TileType`, `LocalPathfinder`, `Zone` đã tồn tại. World tier tạo lớp mới ở trên; Local tier **đổ kết quả vào `GridSystem`** (chỉ cần nâng `new GridSystem(64,64)` → kích thước cấu hình).

---

## 1. Kiến trúc 2 tầng

Giống RimWorld: một **WorldMap** thưa (mỗi ô = một vùng lớn / một "tile thế giới"), và khi player chọn một ô → sinh **LocalMap** dày (nơi thật sự chơi: xây tông môn, combat, di chuyển đệ tử).

```
┌─────────────────────────────────────────────────────────────┐
│  TẦNG WORLD  (Nizhangjie.World)                               │
│  WorldMap: 2D mảng WorldTile[w,h]  (vd 128×128 hoặc 256×256)  │
│  Mỗi WorldTile lưu: elevation, moisture, temp, biome,         │
│     riverFlow, isMountain, lingQiDensity, settlementId, poiId  │
│  → đây là "bản đồ chiến lược": chọn đi đâu, có gì ở đó         │
└───────────────────────────┬─────────────────────────────────┘
                            │  player chọn 1 ô → "bước vào"
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  TẦNG LOCAL  (Nizhangjie.LocalMap — đã có GridSystem)         │
│  GridSystem: Tile[100..200, 100..200]                         │
│  Sinh CHI TIẾT từ thuộc tính của WorldTile cha:               │
│     biome của ô → bảng phối TileType                          │
│     riverFlow > 0 → khắc một con sông xuyên map (nối mép)     │
│     isMountain → cụm núi + vách đá                            │
│     settlement → công trình NPC có sẵn (stamp template)        │
│     poi → ải/di tích/linh mạch                                │
│  → đây là nơi player thật sự xây & chơi                       │
└─────────────────────────────────────────────────────────────┘
```

**Tại sao 2 tầng thay vì một map 200×200 phẳng:** một map đơn 200×200 vẫn là "một địa điểm". Người chơi muốn *nhiều vùng đất, nhiều thành phố, nhiều ải* → cần lớp World thưa để chứa hàng trăm địa điểm, mỗi địa điểm mới phóng to thành 200×200 khi cần. Bộ nhớ chỉ giữ 1 LocalMap active tại một thời điểm (xem mục 10).

---

## 2. Dữ liệu cần thêm

### 2.1. Mở rộng `TileType` (LocalMap/Grid/Tile.cs)

`TileType` hiện có 8 giá trị (Grass/Stone/Water/Road/Mountain/Forest/Sacred/Swamp). Để map chân thực cần thêm. **Giữ giá trị cũ, append để không vỡ save/JSON:**

```csharp
public enum TileType
{
    Grass = 0, Stone = 1, Water = 2, Road = 3, Mountain = 4,
    Forest = 5, Sacred = 6, Swamp = 7,
    // ── M10 thêm ───────────────────────────────
    Sand        = 8,   // bãi cát / sa mạc — passable, không buildable tốt
    Dirt        = 9,   // đất trống / đường mòn
    River       = 10,  // sông (nước chảy) — impassable trừ khi có cầu
    ShallowWater= 11,  // nước nông — passable cost cao (lội)
    Snow        = 12,  // tuyết — passable cost cao
    Ice         = 13,  // băng — passable trơn (cost thấp, risk)
    Lava        = 14,  // dung nham — impassable + hazard (tie vào CombatTerrain)
    Marsh       = 15,  // bùn lầy nặng
    Ruins       = 16,  // nền di tích — buildable, có thể có loot
    Farmland    = 17,  // đất canh tác (đã cày)
    SpiritVein  = 18,  // lộ thiên linh mạch — LingQiModifier cao
    Cliff       = 19,  // vách đá — impassable, viền núi
}
```

> Cập nhật `GridSystem.IsPassableType()` và `LocalPathfinder.GetMoveCost()` cho các type mới. Đề xuất cost: River/Lava/Cliff = ∞; ShallowWater/Marsh = 4; Snow = 2.5; Ice = 0.8; Sand = 1.5; Dirt = 1.0; Farmland = 1.0; Ruins/SpiritVein = 1.0.

### 2.2. Class & enum mới — file tree đề xuất

```
World/
├── WorldTile.cs              — plain data: elevation, moisture, temp, biome, riverFlow, ...
├── WorldMap.cs               — WorldTile[,] + accessor (giống GridSystem nhưng tầng world)
├── BiomeType.cs              — enum biome (Plains, Forest, Mountain, Desert, Tundra, Swamp, ...)
├── Generation/
│   ├── WorldGenConfig.cs     — POCO load từ JSON (seed, size, noise params, ngưỡng biome...)
│   ├── WorldGenerator.cs     — orchestrator: chạy lần lượt các IWorldGenPass
│   ├── IWorldGenPass.cs      — interface 1 bước pipeline: void Run(WorldGenContext ctx)
│   ├── WorldGenContext.cs    — state dùng chung giữa các pass (map, rng, buffers)
│   └── Passes/
│       ├── ElevationPass.cs      — fBm noise → elevation [0..1]
│       ├── ClimatePass.cs        — moisture + temperature noise
│       ├── BiomePass.cs          — phân loại biome (Whittaker)
│       ├── MountainPass.cs       — ridged noise → dãy núi
│       ├── HydrologyPass.cs      — sông (flow accumulation) + hồ + mực biển
│       ├── RegionPass.cs         — Voronoi → vùng/lãnh thổ phe phái
│       ├── SettlementPass.cs     — chấm điểm & đặt thành phố/tông môn NPC
│       ├── RoadPass.cs           — A* nối các settlement
│       ├── PoiPass.cs            — Poisson-disk đặt ải/dungeon/di tích/linh mạch
│       └── NamingPass.cs         — đặt tên vùng/thành/sông (bảng tên Hán-Việt)
├── Noise/
│   ├── FastNoise.cs          — Perlin/Simplex + fBm + ridged + domain warp (deterministic)
│   └── PoissonDisk.cs        — sampling rải điểm cách đều
├── Pois/
│   ├── PoiType.cs            — enum: Dungeon, Ruins, SpiritVein, SecretRealm(ải), BanditCamp...
│   └── PointOfInterest.cs    — data 1 POI trên world
└── LocalGen/
    ├── LocalMapGenerator.cs  — sinh GridSystem chi tiết từ 1 WorldTile
    ├── BiomeTilePalette.cs   — biome → trọng số TileType (load JSON)
    ├── RiverCarver.cs        — khắc sông xuyên local map, khớp mép với neighbor
    ├── StructureStamper.cs   — "dán" template công trình NPC vào grid
    ├── StructureTemplate.cs  — POCO mẫu công trình (mảng ký tự → TileType + occupant)
    └── ResourceScatter.cs    — rải linh mạch / quặng / rừng / di tích trên local map
```

### 2.3. JSON definitions mới (load qua `JsonDataLoader`)

- `Data/Definitions/BiomeDefinition.cs` — biome id, ngưỡng (elevation/moisture/temp), bảng trọng số TileType cho local, danh sách POI cho phép, tần suất tài nguyên, multiplier linh khí.
- `Data/Definitions/WorldGenConfig` (đã liệt kê) — toàn bộ tham số sinh.
- `Data/Definitions/StructureTemplateDefinition.cs` — mẫu công trình (grid ký tự + bảng ánh xạ + danh sách building cần đặt qua `BuildingDefinition` có sẵn).
- `Data/Definitions/PoiDefinition.cs` — loại POI, biome hợp lệ, mật độ, reward tier (nối vào hệ Quest M4).

---

## 3. Mô hình Seed & Determinism (ràng buộc cứng)

Cùng seed → cùng thế giới, **và** cùng một ô world luôn cho ra cùng một LocalMap (kể cả khi đóng/mở lại). Không lưu toàn bộ 40.000 tile của mọi LocalMap — chỉ lưu *thay đổi của player* (delta) trên các map đã ghé.

```
worldSeed                       (int, người chơi nhập hoặc random)
└─ mỗi pass dùng RNG riêng:  passRng = new System.Random(Hash(worldSeed, passId))
└─ localSeed(tileX,tileY) = Hash(worldSeed, tileX, tileY)
       → LocalMapGenerator dùng localSeed → map chi tiết tái lập 100%
```

- Dùng `System.Random` riêng cho từng pass / từng tile (KHÔNG dùng `UnityEngine.Random` global — không deterministic xuyên frame).
- `Hash(...)` = một hàm trộn ổn định (vd FNV-1a hoặc xxHash trên các int) → tránh tương quan giữa các seed gần nhau.
- Noise (`FastNoise`) seed bằng `worldSeed`; offset toạ độ theo `tileX*localSize`, `tileY*localSize` để **mép local map khớp liền** với ô bên cạnh (xem mục 9.2 — edge stitching).

Lưu game = `worldSeed` + danh sách `WorldTile` đã bị player sửa (settlement đã xây) + delta tile của mỗi LocalMap đã ghé. Map chưa ghé → tái sinh on-demand từ seed.

---

## 4. TẦNG WORLD — pipeline tổng quan

`WorldGenerator.Generate(config)` chạy danh sách `IWorldGenPass` **theo đúng thứ tự** dưới đây, mỗi pass ghi vào `WorldGenContext` (chứa `WorldMap` + buffer float). Mỗi pass đọc output của pass trước.

```
Pass 0  WorldGenConfig + seed         → khởi tạo WorldMap rỗng, các RNG
Pass 1  ElevationPass                 → elevation[x,y] ∈ [0..1]
Pass 2  ClimatePass                   → moisture[x,y], temperature[x,y]
Pass 3  MountainPass                  → đánh dấu isMountain + nâng elevation theo ridged noise
Pass 4  HydrologyPass                 → seaLevel, lakes, riverFlow[x,y] (sông chảy từ núi ra biển)
Pass 5  BiomePass                     → biome[x,y] (sau khi đã biết nước & núi)
Pass 6  RegionPass                    → regionId[x,y] (Voronoi) = lãnh thổ phe
Pass 7  SettlementPass                → đặt thành phố / tông môn NPC (suitability)
Pass 8  RoadPass                      → đường nối settlement (A* trên world cost)
Pass 9  PoiPass                       → ải/dungeon/di tích/linh mạch (Poisson-disk theo biome)
Pass 10 NamingPass                    → tên vùng, thành, sông, núi
```

> Lưu ý thứ tự: **núi & nước tính trước biome**, vì biome phụ thuộc độ cao + độ ẩm + có gần nước không. Sông tính sau núi (sông chảy *xuống* từ đỉnh núi). Đây là điểm dễ làm sai nhất.

### 4.1. `WorldTile` — các trường

```csharp
public sealed class WorldTile
{
    public int X, Y;
    public float Elevation;     // 0=đáy biển, 1=đỉnh núi
    public float Moisture;      // 0=khô, 1=ẩm
    public float Temperature;   // 0=lạnh, 1=nóng
    public BiomeType Biome;
    public float RiverFlow;     // 0=không sông; >0 = lưu lượng (vẽ sông to/nhỏ)
    public bool  IsMountain;
    public bool  IsWater;       // biển/hồ
    public float LingQiDensity; // mật độ linh khí vùng (ảnh hưởng local SpiritVein)
    public int   RegionId;      // -1 nếu chưa gán
    public string SettlementId; // null nếu không có
    public string PoiId;        // null nếu không có
    public bool  HasRoad;
}
```

---

## 5. Pass 1–2: Elevation & Climate (noise)

### 5.1. Elevation — fBm + domain warp + island falloff

```
elevation(x,y):
    nx, ny = (x,y) / worldSize * frequency
    # domain warp cho bờ biển/dãy núi tự nhiên hơn (không quá tròn)
    wx = nx + warpAmp * noise(nx + 5.2, ny + 1.3)
    wy = ny + warpAmp * noise(nx + 9.1, ny + 4.7)
    e  = fBm(wx, wy, octaves=5, lacunarity=2.0, gain=0.5)   # ∈ [0..1]
    # tuỳ chọn island/continent mask: kéo mép map xuống biển
    if config.islandFalloff:
        d = distanceToEdgeNormalized(x,y)        # 0 ở rìa, 1 ở giữa
        e *= smoothstep(0, 1, d)                 # rìa thành biển
    return clamp01(e)
```

- `fBm` = cộng nhiều octave Perlin/Simplex (`FastNoise`). 5 octave đủ cho 256×256.
- `domain warp` (làm méo toạ độ trước khi lấy noise) là thủ thuật rẻ tiền cho địa hình "có hồn", tránh kiểu blob tròn.
- `islandFalloff` quyết định thế giới là lục địa giữa biển hay đất liền nối liền — đưa vào config.

### 5.2. Climate — moisture & temperature

```
temperature(x,y):
    base = 1 - abs(y - worldHeight/2) / (worldHeight/2)   # xích đạo nóng, cực lạnh (theo trục Y)
    base -= elevation(x,y) * lapseRate                     # càng cao càng lạnh
    base += tempNoise(x,y) * 0.1                           # nhiễu nhẹ
moisture(x,y):
    m = fBm(x,y, octaves=4, seedOffset=MOISTURE)
    m += proximityToWater(x,y) * coastalMoistureBoost      # gần biển/hồ ẩm hơn
    # tuỳ chọn rain-shadow: phía khuất gió của núi thì khô (nâng cao về sau)
```

→ Lưu `Moisture`, `Temperature` vào mỗi `WorldTile`. (Có thể tính `proximityToWater` sau Hydrology rồi quay lại; hoặc xấp xỉ bằng "elevation thấp" ở pass này rồi tinh chỉnh — đề xuất: tính sơ bộ ở đây, refine moisture ven sông trong HydrologyPass.)

---

## 6. Pass 3: Mountain (ridged noise)

Núi là "xương sống" của bản đồ → tách riêng để kiểm soát hình dạng dãy.

```
mountainMask(x,y):
    r = ridgedNoise(x,y, octaves=4)          # ridged = 1 - |perlin|, tạo sống núi sắc
    if elevation(x,y) > mountainElevThreshold AND r > ridgeThreshold:
        tile.IsMountain = true
        tile.Elevation = max(elevation, lerp(elevation, 1.0, r))   # đẩy cao thêm
```

- `ridgedNoise` cho ra các sống dài, liên tục → trông như dãy núi thật, không phải đốm rời rạc.
- Núi cao = nguồn của sông (Hydrology dùng `Elevation` đã được Mountain nâng).
- Mảng `IsMountain` này sẽ map sang `TileType.Mountain`/`Cliff` ở tầng local.

---

## 7. Pass 4: Hydrology — sông, hồ, biển

Đây là pass khó nhất và "ăn tiền" nhất về độ chân thực. Thuật toán cổ điển: **droplet / flow accumulation**.

```
# 7.1 Mực nước biển
seaLevel = config.seaLevel            # vd 0.30
for each tile: tile.IsWater = (elevation < seaLevel)

# 7.2 Hồ: vùng trũng kín (local minima) dưới ngưỡng lake → đánh dấu IsWater
fillDepressions()                     # (priority-flood) tránh sông "kẹt" trong hố

# 7.3 Sông: flow accumulation
flow[x,y] = 1 cho mọi tile (mưa đều)
sort tiles giảm dần theo elevation
for tile theo thứ tự cao→thấp:
    d = neighbor thấp nhất (8-dir)
    flow[d] += flow[tile]             # nước dồn xuống hạ lưu
for each tile:
    if flow[tile] > riverThreshold AND not IsWater:
        tile.RiverFlow = flow[tile]   # thành sông; flow lớn = sông to
        # đi theo dòng xuống tới biển/hồ → đánh dấu cả tuyến
```

- `fillDepressions` (priority-flood, Barnes 2014) bắt buộc nếu không sông sẽ kết thúc lửng giữa map.
- `riverThreshold` điều khiển có nhiều sông nhỏ hay ít sông lớn.
- `RiverFlow` truyền xuống tầng local: ô world có sông → local map khắc một con sông; độ rộng tỉ lệ `log(RiverFlow)`.
- Refine moisture: tile cạnh sông tăng `Moisture` → BiomePass cho ra dải rừng/đồng bằng ven sông.

---

## 8. Pass 5: Biome classification (Whittaker)

Sau khi đã biết elevation, nước, núi, nhiệt độ, độ ẩm → phân loại biome bằng bảng tra (mô hình Whittaker: trục nhiệt × ẩm). Toàn bộ ngưỡng nằm trong `BiomeDefinition` JSON, **không hard-code**.

```
biome(x,y):
    if tile.IsWater:            return Ocean / Lake
    if tile.IsMountain:         return Mountain (hoặc SnowPeak nếu temp thấp)
    # còn lại tra bảng theo (temperature, moisture):
    cold  + dry   → Tundra
    cold  + wet   → Taiga (rừng lá kim, tuyết)
    temp  + dry   → Plains/Grassland
    temp  + wet   → Forest (rừng linh thú)
    hot   + dry   → Desert
    hot   + wet   → Swamp/Jungle (đầm lầy linh chướng)
    elevation cao vừa → Highlands/Rocky
```

`BiomeType` đề xuất: `Ocean, Lake, Beach, Plains, Forest, Taiga, Jungle, Swamp, Desert, Savanna, Tundra, Highlands, Mountain, SnowPeak, SpiritLand(linh địa hiếm)`.

> `SpiritLand` = biome đặc biệt, hiếm, `LingQiDensity` rất cao → bản đồ local sẽ đầy `SpiritVein`/`Sacred`. Đặt thủ công ở vài ô có noise linh khí cao nhất (gắn với `Sacred` TileType & cơ chế tu luyện M4).

---

## 9. Pass 6–10: Region, Settlement, Road, POI, Naming

### 9.1. RegionPass — lãnh thổ (Voronoi)
Rải N điểm "thủ phủ" (Poisson-disk), gán mỗi tile về điểm gần nhất → `RegionId`. Mỗi region = một phe/lãnh thổ (nối vào `FactionDefinition`/`Sect` NPC). Cho phép biên giới tự nhiên bằng cách cộng nhiễu vào khoảng cách Voronoi.

### 9.2. SettlementPass — thành phố & tông môn NPC (suitability scoring)
Không đặt thành phố ngẫu nhiên. Chấm điểm mỗi tile:

```
score(tile) =  w1 * gầnNước(tile)            # cạnh sông/hồ/biển
             + w2 * đấtBằng(1 - elevationDốc)
             + w3 * linhKhí(LingQiDensity)     # tu tiên → ưu tiên linh khí
             + w4 * gầnTàiNguyên(núi/rừng gần)
             - w5 * quáGầnSettlementKhác       # giãn cách tối thiểu
chọn top-K tile theo score, mỗi region ≥1 thành → gán SettlementId
```

Phân loại: thành phố lớn / thị trấn / thôn / **tông môn NPC** (đối thủ — nối `CompetitorMarketPool`, `Sect` M4-M7). Mỗi settlement lưu tier để tầng local biết stamp template lớn hay nhỏ.

### 9.3. RoadPass — đường nối
Dùng lại tư tưởng A* (như `LocalPathfinder` nhưng trên `WorldMap`): cost = đi qua núi/sông đắt, đồng bằng rẻ. Nối các settlement theo cây khung tối thiểu (MST) + vài cạnh phụ → mạng đường thực tế. Tile trên đường: `HasRoad = true` → local map sẽ có `TileType.Road` ở mép tương ứng.

### 9.4. PoiPass — ải / dungeon / di tích / linh mạch
Poisson-disk rải POI sao cho không chồng settlement, lọc theo biome hợp lệ (`PoiDefinition.allowedBiomes`):

- `SecretRealm` (bí cảnh/ải) — instance combat lớn, reward tier cao.
- `Dungeon` (động phủ) — nối hệ Quest `Dungeon`/`RequiresCombat` M4.
- `Ruins` (di tích cổ) — local map có cụm `TileType.Ruins` + loot.
- `SpiritVein` (linh mạch lộ thiên) — local map có nhiều `SpiritVein`, buff tu luyện.
- `BanditCamp` (sơn tặc) — nối Quest `Bandit`/`Punitive`.

### 9.5. NamingPass — đặt tên
Bảng tên Hán-Việt (giống `DiscipleNameTable` đã có): núi ("X Sơn"), sông ("X Giang/Hà"), vùng ("X Vực/Châu"), thành ("X Thành"). Ghép tố + hậu tố theo biome (vd biome lạnh → "Hàn", "Tuyết"; linh địa → "Linh", "Tiên").

---

## 10. TẦNG LOCAL — sinh chi tiết từ 1 WorldTile

Khi player chọn ô `(tx, ty)` để vào, `LocalMapGenerator.Generate(worldTile, localSeed, size)` trả về một `GridSystem` đã sơn đầy đủ. **Đây là nơi đổ vào hạ tầng GridSystem hiện có.**

```
LocalMapGenerator.Generate(worldTile, localSeed, size=200):
  rng = new Random(localSeed)
  grid = new GridSystem(size, size)

  # 10.1 Nền theo biome
  palette = BiomeTilePalette.For(worldTile.Biome)   # trọng số TileType từ JSON
  detailNoise = fBm seeded bằng localSeed, offset = (tx*size, ty*size)  # KHỚP MÉP
  for each (x,y):
      n = detailNoise(x,y)
      grid.SetTileType((x,y), palette.Pick(n, rng)) # vd forest: 60% Forest,30% Grass,10% Dirt

  # 10.2 Độ cao cục bộ → núi/vách (nếu worldTile.IsMountain hoặc giáp ô núi)
  MountainLocal: vùng elevation local cao → Mountain/Cliff/Stone

  # 10.3 Sông (nếu worldTile.RiverFlow > 0)
  RiverCarver.Carve(grid, worldTile, neighbors, rng)
      # khắc dải River + ShallowWater, ĐIỂM VÀO/RA khớp với ô world kế bên
      width = clamp(log(RiverFlow))

  # 10.4 Rải tài nguyên
  ResourceScatter:
      SpiritVein tiles  ~ worldTile.LingQiDensity   (gán Tile.LingQiModifier > 0)
      quặng (Stone cluster) gần biome núi
      rừng linh thú (Forest cluster) theo moisture
      Swamp/Marsh nếu biome đầm lầy (Tile.LingQiModifier < 0)

  # 10.5 Công trình có sẵn
  if worldTile.SettlementId != null:
      StructureStamper.Stamp(grid, settlementTemplate, rng)   # thị trấn NPC: nhà, đường, chợ
  if worldTile.PoiId != null:
      StructureStamper.Stamp(grid, poiTemplate, rng)          # di tích/ải

  # 10.6 Zone & validate
  tạo Zone gợi ý (Living/Farming...) quanh khu bằng phẳng
  ValidateConnectivity(grid)   # flood-fill: đảm bảo có vùng buildable đủ lớn & tới được
  return grid
```

### 10.1. `BiomeTilePalette` (JSON-driven)
Mỗi biome → danh sách `(TileType, weight)` + ngưỡng noise. Ví dụ JSON:

```json
{
  "biome": "Forest",
  "lingQiBase": 0.2,
  "tileWeights": [
    { "type": "Forest", "weight": 0.55 },
    { "type": "Grass",  "weight": 0.30 },
    { "type": "Dirt",   "weight": 0.10 },
    { "type": "Stone",  "weight": 0.05 }
  ],
  "allowedResources": ["SpiritVein", "OreCluster"],
  "structureTemplates": ["village_small", "hermit_hut"]
}
```

### 10.2. Edge stitching (khớp mép — chống "đứt" giữa các ô)
- Detail noise của local map dùng **toạ độ thế giới tuyệt đối** `(tx*size + x, ty*size + y)` → mép phải của ô `(tx,ty)` trùng mép trái của ô `(tx+1,ty)`.
- Sông: `RiverCarver` đọc `RiverFlow` của 4 ô world kế bên → xác định **cạnh nào sông chảy vào/ra**, ép điểm vào/ra nằm đúng vị trí xác định bởi `Hash(worldSeed, edge)` → hai map liền nhau nối sông liền mạch.
- Đường (`HasRoad`): tương tự, điểm vào/ra đường nằm ở giữa cạnh giáp ô có road.

### 10.3. `StructureStamper` & `StructureTemplate`
Template = lưới ký tự + bảng ánh xạ (như tilemap text), kèm danh sách building thật cần `GridSystem.PlaceBuilding()`:

```
template "village_small" (12×12):
  "............"
  ".RRRRRRRRRR."     R = Road
  ".R.HH..HH.R."     H = nhà NPC (occupant)
  ".R.HH..HH.R."     M = chợ (market — nối Market/Shop M4)
  ".R...MM...R."     . = Grass
  ...
stamp: chọn vùng bằng phẳng đủ rộng (dùng ValidateConnectivity), xoay/lật ngẫu nhiên theo rng,
       set TileType + gọi PlaceBuilding cho từng occupant qua BuildingDefinition có sẵn.
```

POI/di tích cũng là template (cụm `Ruins`, có thể có "loot tile" gắn quest).

### 10.4. ValidateConnectivity
Sau khi sơn xong: flood-fill từ một tile buildable trung tâm; nếu vùng đi-được/xây-được < ngưỡng (vd 25% map) → đục bớt núi/sông hoặc reroll cụm chướng ngại. Đảm bảo player luôn có chỗ xây tông môn & đường ra. Dùng lại `LocalPathfinder` để kiểm tra cổng vào nối tới khu trung tâm.

---

## 11. Hiệu năng (200×200 = 40.000 tile)

- 40k tile **không nặng** với plain C# nếu chỉ chạy 1 lần lúc vào map. Mục tiêu < ~200ms.
- Tránh `LINQ`/allocation trong vòng lặp noise nóng; dùng mảng `float[]` phẳng, index = `y*size + x`.
- Nếu thấy giật khi vào map: chạy `LocalMapGenerator` trên **background thread** (plain C#, không đụng Unity API trong lúc gen — chỉ build dữ liệu `Tile`/`TileType`), rồi apply visual trên main thread. (Mép tilemap/sprite dựng ở UI layer, không trong generator — đúng nguyên tắc Tile.cs hiện tại.)
- WorldMap 256×256 = 65k tile, gen 1 lần lúc tạo save → chấp nhận chờ vài trăm ms với màn loading.
- Chỉ giữ **1 LocalMap active**. Map đã ghé: lưu *delta* (thay đổi của player) thay vì toàn bộ tile. Map chưa ghé: tái sinh từ `localSeed` khi cần (deterministic ⇒ giống hệt lần trước).
- (Tuỳ chọn nâng cao sau) Unity Job System + Burst cho noise nếu cần map > 512 hoặc gen realtime nhiều ô. Không cần cho M10.1.

---

## 12. Thứ tự triển khai (milestones)

Chia nhỏ để mỗi bước test được độc lập (giống cách M1–M9 đã làm):

```
M10.1  Hạ tầng noise + World data
        FastNoise (Perlin/fBm/ridged + domain warp), PoissonDisk,
        WorldTile, WorldMap, BiomeType, WorldGenConfig (JSON load).
        ✅ Test: noise deterministic theo seed; fBm trả [0..1].

M10.2  World terrain passes
        ElevationPass → ClimatePass → MountainPass → HydrologyPass → BiomePass.
        ✅ Test: dump WorldMap ra ảnh/CSV trong Editor; mắt thường thấy lục địa+núi+sông+biome hợp lý.

M10.3  World content passes
        RegionPass (Voronoi) → SettlementPass (suitability) → RoadPass (A*) →
        PoiPass (Poisson) → NamingPass.
        ✅ Test: số settlement đúng config; mọi settlement nối đường; POI không chồng settlement.

M10.4  Mở rộng TileType + cập nhật GridSystem/LocalPathfinder cho type mới.
        ✅ Test: passable/cost đúng cho River/Cliff/Snow/...

M10.5  LocalMapGenerator nền + biome palette + edge stitching.
        Nâng LocalMapManager: new GridSystem(configSize) thay 64×64.
        ✅ Test: cùng (tx,ty)+seed → grid giống hệt; mép 2 ô liền khớp noise.

M10.6  RiverCarver + ResourceScatter (sông liền mạch, linh mạch theo LingQiDensity).
        ✅ Test: ô world có RiverFlow>0 ⇒ local có dải River nối 2 mép đúng.

M10.7  StructureStamper + StructureTemplate + PreBuilt settlements/POI.
        Nối building NPC vào BuildingDefinition/Market/Shop có sẵn.
        ✅ Test: stamp village ⇒ PlaceBuilding thành công, đường nối tới mép.

M10.8  ValidateConnectivity + tinh chỉnh + lưu/đọc delta (persistence).
        ✅ Test: 100 ô random đều có vùng buildable ≥ ngưỡng & tới được.
```

---

## 13. Checklist test (Editor, không cần Play Mode — theo style Tests/Editor/M*Test.cs)

- [ ] `FastNoise` cùng seed → cùng output (determinism).
- [ ] `Hash(worldSeed, tx, ty)` phân tán tốt, không tương quan ô kề.
- [ ] HydrologyPass: mọi sông kết thúc ở biển/hồ (không "treo" giữa đất).
- [ ] BiomePass: phân bố biome khớp bảng ngưỡng JSON (đếm tỉ lệ).
- [ ] SettlementPass: giãn cách tối thiểu được tôn trọng; mỗi region ≥ 1 settlement.
- [ ] RoadPass: đồ thị đường liên thông (mọi settlement reachable).
- [ ] Edge stitching: tile mép phải ô A == tile mép trái ô B (noise & sông & đường).
- [ ] LocalMapGenerator: cùng input → grid byte-identical.
- [ ] ValidateConnectivity: vùng buildable ≥ ngưỡng trên mẫu lớn ô.
- [ ] StructureStamper: không đặt building lên tile impassable; footprint hợp lệ.

---

## 14. Tóm tắt thuật toán dùng cho từng mục tiêu

| Mục tiêu | Thuật toán | Pass |
|---|---|---|
| Địa hình lục địa tự nhiên | fBm Perlin/Simplex + domain warp + island falloff | Elevation |
| Khí hậu | gradient theo vĩ độ + lapse rate theo độ cao + noise | Climate |
| Dãy núi sắc nét | ridged multifractal noise | Mountain |
| Sông chảy đúng vật lý | fill-depressions + flow accumulation | Hydrology |
| Hồ / biển | sea level threshold + priority-flood | Hydrology |
| Biome đa dạng | Whittaker (nhiệt × ẩm) tra bảng JSON | Biome |
| Lãnh thổ phe | Voronoi + nhiễu biên | Region |
| Đặt thành phố hợp lý | suitability scoring + min-distance | Settlement |
| Đường xá | A* + MST trên world cost | Road |
| Rải ải/dungeon/linh mạch | Poisson-disk + lọc biome | POI |
| Sông/đường khớp mép local | toạ độ noise tuyệt đối + hash điểm vào/ra | LocalGen |
| Công trình có sẵn | stamp template (lưới ký tự) + PlaceBuilding | LocalGen |
| Đảm bảo chơi được | flood-fill connectivity + reroll cục bộ | LocalGen |

---

## 15. Điểm nối với hệ thống đã có (đừng làm lại)

- `GridSystem` / `Tile` / `TileType` — **đích đến** của LocalMapGenerator. Chỉ append TileType + nâng kích thước. Không viết grid mới.
- `LocalPathfinder` — dùng lại cho ValidateConnectivity & cho world RoadPass (clone tư tưởng A*).
- `Zone` / `ZoneType` — LocalMapGenerator tự sinh zone gợi ý.
- `BuildingDefinition` / `BuildingController` / `Market` / `Shop` — settlement NPC stamp dùng các định nghĩa này.
- `Sect` / `CompetitorMarketPool` (M6) / `FactionDefinition` — region & tông môn NPC nối vào đây.
- `Quest` / `QuestTypes` / `DispatchSystem` (M4) — POI (Dungeon/Bandit/SecretRealm) sinh quest tương ứng.
- `JsonDataLoader` / `DataRegistry<T>` — load mọi `*Definition` mới.
- `EventBus` — `WorldGenerator` publish `WorldGeneratedEvent`; `LocalMapGenerator` publish `LocalMapReadyEvent` (thêm vào `SampleEvents.cs`).
- `DiscipleNameTable` — mẫu cho `NamingPass` (bảng tên Hán-Việt).
- `LingQiModifier` (Tile) + `Sacred`/`SpiritVein` — kênh truyền linh khí từ world `LingQiDensity` xuống local.

---

*Tài liệu kế hoạch M10 — sẵn sàng để bắt đầu từ M10.1. Mỗi milestone độc lập, test được trong Editor, tuân thủ JSON-first + plain-C# + deterministic.*
