# CODE SYSTEM RULE — Nizhangjie

> **Đây là bộ luật, không phải hướng dẫn.** Mỗi luật dưới đây sinh ra từ một con bug THẬT đã xảy ra
> trong chính dự án này — không có luật nào là "best practice chép về". Chỗ nào ghi *bug thật* là chỗ đó
> đã tốn thời gian sửa một lần rồi.
>
> Bản đồ code / số liệu chi tiết → `CLAUDE.md`. File này chỉ trả lời **"được phép làm gì, cấm làm gì"**.

---

## PHẦN A — LUẬT KIẾN TRÚC (không thương lượng)

### A1. JSON-first
Data khai trong JSON, nạp qua `JsonDataLoader` + `DataRegistry<T>`. **Không tạo ScriptableObject mới cho
data.** Content mới = thêm file JSON, không sửa code.

*Phép thử:* modder thêm một biome / một trang bị / một công thức mới **mà không mở Visual Studio** — nếu
không được, chỗ đó chưa data-driven.

### A2. Không singleton
Không `Instance`. Lấy dependency qua `ServiceLocator` hoặc constructor injection. Ưu tiên constructor —
`ServiceLocator` là dependency ẩn, dùng nhiều thì không ai biết hệ nào cần gì.

### A3. EventBus chỉ cho 1-nhiều
3 scope: `GlobalEventBus` / `CombatEventBus` / `LocalMapEventBus`.
**"A xong → B chạy" thì gọi thẳng B**, đừng bắn event. Event là để *thông báo cho những ai quan tâm*,
không phải để nối dây điều khiển.

⚠ Priority: **số CAO chạy TRƯỚC**. Handler muốn chặn (`Cancel`) phải để priority cao hơn handler muốn bị
chặn. EventBus **không tự bỏ qua** event đã cancel — mỗi handler tự kiểm `e.IsCancelled`.

### A4. Deterministic
Cùng seed → cùng kết quả. Cấm để thứ tự duyệt `Dictionary`/`HashSet` lọt vào logic gen. Registry nào cần
duyệt thì giữ thêm `_ordered` (mẫu: `HealthRegistry`, `MoodRegistry`, `CultivationRegistry`).

### A5. LÕI ⟂ CONTENT-PACK *(M37, 2026-08-15 — luật mới, quan trọng nhất)*

**Dự án này là ENGINE tạo thế giới. Xianxia chỉ là MỘT content-pack.**

- Hệ thống mới nhận `Core.Pawns.PawnBody`, **KHÔNG** nhận `Disciple`.
- Cần tu vi thì **hỏi** `Sim.Xianxia.CultivationRegistry` — trả `null` nghĩa là phàm nhân, và đó là
  trạng thái **hợp lệ**, không phải thiếu dữ liệu.
- Lõi chừa **chỗ trống** cho pack điền (`PawnBody.HomProvider/HomApplier`), lõi **không tự điền**.
- `Core/` **cấm** phụ thuộc `Sim/`. Ngoại lệ duy nhất: `Core/Bootstrap/GameRoot.cs` (composition root —
  vai trò của nó là biết mọi tầng để nối chúng lại).

> **PHÉP THỬ SỐNG-HAY-CHẾT:** xoá `Sim/Xianxia/` + bỏ dòng `XianxiaContentPack.Install()` → game **vẫn
> phải chạy đủ**: vẫn đói, mệt, suy sụp, làm việc, đánh nhau, chết. Chỉ khác: ai cũng phàm nhân.
> Ngày nào điều đó sai, nghĩa là có người đã kéo nội dung ngược vào lõi.

**Ranh giới CƠ CHẾ ⟂ NỘI DUNG:** cơ chế học kỹ năng là lõi, *"Cửu Dương Thần Công"* là nội dung. Cơ chế
chế tác là lõi, *công thức đan dược* là nội dung. Cơ chế trang bị là lõi, *ô Pháp Bảo* là nội dung.
Bảng phân chia đầy đủ: `CLAUDE.md §5.2`.

### A6. Cấm FORK đường ống — hãy TỔNG QUÁT HOÁ nó
Muốn làm tính năng cho một loại thực thể mới (thú, companion, NPC) mà đường ống hiện tại không nhận, thì
**nới đường ống**, đừng dựng bản sao.

*Bằng chứng cái giá:* Companion (M29) và Beast đã fork hẳn stack AI/di chuyển riêng chỉ để né `Disciple`.
Kết quả: mỗi tính năng phải làm 2–3 lần, và 2 bản luôn lệch nhau dần. M37 sinh ra để trả món nợ đó.

---

## PHẦN B — LUẬT THIẾT KẾ HỆ THỐNG

### B1. Ngưỡng VÀO ≠ ngưỡng RA (vùng chết bắt buộc)
Mọi hệ có ngưỡng chuyển trạng thái **phải** có vùng chết giữa điều kiện vào và điều kiện ra.

*Bug thật (M27):* dùng chung một hàm cho cả "đi ngủ" lẫn "thức dậy" → điểm thức trùng điểm ngủ → vùng
chết = 0 → pawn nhấp nháy `Sleeping/Moving/Training` mỗi 1–2 tick. Nay `MoodConfig.Sanitize()` **ép**
`recoverAt > minorBreakAt`; `ScheduleTest` giữ bất biến vùng chết chống tái phát.

### B2. Một nơi GHI, nhiều nơi ĐỌC
Mỗi mẩu trạng thái có đúng **một** hệ được phép ghi. Ví dụ: `PawnNeedsSystem` chuyển trạng thái suy sụp,
mọi `JobGiver` chỉ đọc `MoodRegistry`. Hệ quả tốt: cơn suy sụp vẫn tiếp diễn kể cả khi pawn đang bận việc
ở Tier khác.

Tương tự: **phần CỘNG thuộc job, phần TRỪ thuộc hệ nhu cầu** — đừng để 2 nơi cùng ghi một con số.

### B3. Enum đã serialize là APPEND-ONLY
`TileType`, `BreakTier`, `BeastState`, `ItemSlot`, `PawnTier`… — chỉ thêm giá trị mới ở cuối. Sửa giá trị
cũ = vỡ mọi save.

Muốn thêm *cơ chế* mới thì thêm **file JSON**, đừng thêm giá trị enum (bài học M34: ô trang bị đã ra JSON,
`ItemSlot` chỉ còn để đọc save cũ).

### B4. Sentinel phải phân biệt "chưa có" với "bằng 0"
`-1f` = "save cũ, chưa có field này". Nhưng **cổng kiểm phải là `>= 0f`, không phải `> 0f`** khi 0 là giá
trị THẬT.

*Ví dụ:* `Hunger = 0` nghĩa là *đói lả nhưng còn sống* — dùng `> 0f` sẽ âm thầm reset về 100 mỗi lần load.
Khác `currentHP` nơi 0 thật sự nghĩa là chết, và `> 0f` ở đó đúng.

### B5. Vòng đời dữ liệu phải khai TƯỜNG MINH
Với mỗi loại thực thể, trả lời rõ: **cái gì mất, khi nào mất, ai xoá.**

*Chuẩn đã chốt (M36):* dữ liệu `Disciple` **không bao giờ** tự mất; token trên map chỉ biến mất qua hành
động tường minh của người chơi ("Bury"). Ngược lại NPC/yêu thú **được phép** tự dọn dữ liệu sau N ngày —
nếu không thì quần thể chỉ phình mãi.

Hai hướng ngược nhau, và **cả hai đều phải viết ra**, đừng để code tự ngầm định.

### B6. Trạng thái na ná nhau phải TÁCH TÊN
Hai khái niệm khác bản chất mà dùng chung một cờ = bug nằm im rất lâu.

*Bug thật (M36):* `IsDowned` (`CurrentHP <= 0`) vẫn đúng **mãi mãi** sau khi chết, vì không ai reset HP.
Code chỉ kiểm `IsDowned` sẽ coi xác vĩnh viễn là "còn cứu được" → pawn đi cứu xác vô thời hạn. Phải thêm
`IsDeceased` tách hẳn.

Tương tự đã chốt: `Loyalty` (thái độ với TẬP THỂ) ⟂ `Mood` (trạng thái cá nhân) — **cố ý không cộng dồn**.

### B7. Lưới an toàn là `ConfigErrors()`, không phải `ResolveReferences()`
`ResolveReferences` chỉ bắt id gõ sai hẳn. Nó **không** bắt được lỗi thiếu tiền tố namespace, vì
`CandidateKeys` tự fallback `core:{id}`.

*Bug thật:* JSON khai `"small_talk"` (thiếu `core:`) → `TryGet` trả null **im lặng** → pawn vẫn đi bộ, vẫn
đứng đủ giờ, nhưng **không bao giờ** áp opinion. Không lỗi, không warning.

Ký tự/ id dùng trong data mà quên khai → `ConfigErrors` phải bắt. Data hỏng → **`Build()` trả null**, KHÔNG
dựng một nửa.

### B8. Cấm quét toàn cục (`FindObjectsByType`/reflection/LINQ-alloc) trong vòng lặp mỗi frame/tick ⚠ QUAN TRỌNG
Mọi lookup chạy trong `Update()`/mỗi tick/mỗi vòng lặp lồng phải là **tra registry đã cache**, không bao
giờ quét lại scene hay Resources tại chỗ. Registry tĩnh (mẫu `WildPlayerToken.All`, tự đăng ký qua
`OnEnable`/`OnDisable`) là cách chuẩn — quét 1 lần lúc khởi tạo, sau đó chỉ đọc.

*Bằng chứng cái giá (2026-08-08, City 874 dân, tua 8x):* `IdleActivityPipeline.ResolveNearbyFurniture`
(M19 Idle Tier 4) gọi `WildPlayerToken.At()` → `FindObjectsByType` **mỗi lần**, nằm trong vòng lặp LỒNG
(mọi món nội thất × mọi ô kề footprint) + `PawnAiBrain.HandleTick` cũng `FindObjectsByType` mỗi tick. Code
viết đúng cho Wild ("pawn thật ít, quét trực tiếp chấp nhận được") nhưng City có 6906 món nội thất — 100×
quy mô Wild. Main Thread 713ms/frame (~1.4 FPS), riêng `JobGiver_StandingOrder` 77ms/lần gọi (97% chi phí
AI). Sau khi đổi sang sổ token tĩnh: 3.26ms/frame, 0.123ms/lần gọi — không phải tối ưu vi mô, là đổi hẳn
độ phức tạp thuật toán.

**Bài học tái dùng — đáng nhớ nhất:** khi port 1 hệ từ map nhỏ (Wild) sang map lớn hơn 100× (City), mọi
chỗ code cũ có ghi/ngầm định *"ít object nên quét trực tiếp chấp nhận được"* PHẢI được rà lại — giả định
đó chỉ đúng ở quy mô cũ. 4 vòng tối ưu dựa trên đọc-code + suy luận (multithread pathfinding, thu hẹp
graph, cache tuyến...) trước đó đều **tối ưu nhầm hệ** — thủ phạm thật chỉ lộ ra khi đo Profiler
([[project_city_commute_perf]], `mcp__UnityMCP__manage_profiler`), xem [[feedback_measure_before_optimizing]].
Nhìn p99/max/số frame >100ms, ĐỪNG nhìn avg — giật là hiện tượng của ĐUÔI, avg vẫn có thể trông đẹp.

### B9. Trạng thái mới → tự hỏi "cái này có sống qua save/load không?"
**Không có gì tự động quét RAM rồi lưu hết.** Thêm field/hệ thống mới mà cần giữ qua lần chơi sau thì phải
tự tay: thêm field vào `SaveData`, viết `SaveSystem/XxxIO.Capture(data)`/`Restore(data)`, rồi gọi cả hai ở
**đúng 3 chỗ** trong `SaveManager` — `WriteFile()` (Capture, dùng chung cho `Save()`+`Create()`), `Load()`
(Restore, đọc save cũ), `Create()` (Restore, ván mới reset về mặc định — thiếu chỗ này thì RAM cũ lẫn vào
ván mới). Quên là **im lặng tuyệt đối**, không log không lỗi, chỉ đơn giản là mất khi tắt game.

*Bug thật:* `DiscipleSaveData` viết kỹ (máu/thương tích/nhu cầu/trang bị) nhưng chưa từng được gọi →
**đệ tử bốc hơi mỗi lần tắt game**. Sử ký (M48) và ký ức (M49) chạy đúng trong game suốt 2 đợt phát triển
mà chưa vào save (`SaveData.chronicle`/`memories` chỉ xuất hiện ở M53) — trong lúc đó, tắt game là **cả
colony mất trí nhớ sạch**.

*Bẫy con khi nối vào save (xem thêm B4/B5):*
- **Ngưỡng save cũ**: list rỗng thường là mặc định hợp lệ ("chưa có gì"); số mà `0` là giá trị THẬT
  (Hunger, Mood...) phải dùng sentinel riêng, đừng lẫn với "field save cũ chưa có".
- **Đừng suy lại** — ghi thẳng trạng thái đã tính (độ tin, tone, breakTier...), đừng cố suy ngược nó ra
  từ dữ liệu khác lúc Restore (mất đúng phần đã lệch khỏi công thức gốc — xem `MoodRegistry.Restore`,
  `MemoryIO.Restore`).
- **Restore lặp qua đường ghi sự kiện bình thường → nhân đôi mỗi vòng save/load.** Lý do
  `WorldChronicle.ReplaceAll` (khác `Record`) tồn tại: nạp lại bằng `Record()` sẽ fan-out xuống
  `MemoryRegistry` lần thứ hai.
- **Thứ tự Restore quan trọng** nếu dữ liệu tham chiếu nhau (sổ quan hệ phải Restore SAU sổ thân xác).

*Phép thử:* gây ra trạng thái đó thật (không phải tưởng tượng), Save, tắt game giả lập (`ActiveSave=null`
hoặc mở lại project), Load — trạng thái phải khớp **tuyệt đối**, không về mặc định.

### B8. Asmdef giữ ranh giới GIỮA assembly — TEST giữ ranh giới BÊN TRONG một assembly
*(2026-10-12: trước đó cả dự án là MỘT assembly `Assembly-CSharp`, layering thuần quy ước.)*

Assembly hiện có: **`Nizhangjie`** (toàn bộ `Scripts/`, runtime) · **`Nizhangjie.Editor`** (`Scripts/Editor/` + mọi thư mục `Editor/` khác trong
`Scripts/` nối vào bằng file `Nizhangjie.Editor.asmref`) · **`Nizhangjie.VoxelView`** (+ `.Editor`) — lớp vẽ voxel 3D, tham chiếu
`Nizhangjie`; **game KHÔNG tham chiếu được nó** (compiler chặn). `Scripts/AssemblyInfo.cs` mở `internal` cho test/editor/lớp vẽ.

- Thư mục `Editor/` MỚI trong `Scripts/` ⇒ thêm `Nizhangjie.Editor.asmref` (chép từ thư mục Editor khác). Thiếu là code editor bị dịch vào
  assembly runtime: Editor vẫn chạy, nhưng bản build vỡ.
- Code dò assembly theo TÊN phải biết tên mới: assembly chỉ-Editor nay đuôi `.Editor` (không chỉ `-Editor`). *Bug thật ngay khi đổi:*
  `CombatDriverRegistry` chỉ lọc `-Editor` ⇒ contributor giả của test lọt vào lúc chơi thật.
- Nút/sự kiện UI nối qua Inspector lưu `m_TargetAssemblyTypeName: Kiểu, TênAssembly` trong scene/prefab (YAML có thể XUỐNG DÒNG giữa kiểu và
  tên assembly). Dời kiểu sang assembly khác ⇒ phải sửa chuỗi đó, nếu không listener im lặng không chạy.

Bên trong `Nizhangjie` (Core ⟂ Sim, sim ⟂ renderer 2D…) layering vẫn là quy ước ⇒ ranh giới đó phải có **test canh**:
`Arch_CoreNeverDependsOnSim` quét toàn bộ `Scripts/Core/`; `Run Render Separation Tests` canh lớp vẽ.
⚠ Chốt kiểu này phải bắt **cả hai cách viết** — `using Nizhangjie.Sim.*` LẪN tên đầy đủ `Sim.X.` trong
code. (Bản đầu của chính chốt này chỉ bắt `using` và đã bỏ lọt.)

### B10. Sim là dữ liệu C# thuần — tick không đụng Unity, lớp vẽ chỉ chép theo *(2026-10-17)*
**Mục tiêu:** để một ngày nào đó dời sim sang luồng riêng (sim khựng thì khung hình vẫn mượt) mà **không phải viết lại lõi**. Unity chỉ
cho đụng `Transform`/`GameObject` trên luồng chính, nên hệ nào để trạng thái sim nằm trong scene là hệ đó khoá vĩnh viễn vào luồng chính.
Luật này KHÔNG làm việc dời đó thành miễn phí: còn nợ di chuyển (mục 6). Áp cho **code mới và HÀM đang sửa** — không bắt dọn cả file mỗi
lần đụng vào, không bắt rà lại code cũ một lượt.

1. **Trạng thái sim sống trên POCO/struct**, không trên `Transform` hay field chỉ lớp vẽ đọc: vị trí, đường đi, máu, việc đang làm, mục
   tiêu. `Transform` là **bản chép** để vẽ, không phải nguồn sự thật. Mẫu đúng: `BeastView.pos` → `BeastBehaviour.SyncView`;
   `WildPlayerToken.CellPos`; `NpcRecord.Position` do `CityNpcRenderer` đọc (chạy SAU `CityNpcSystem`).
2. **Code tick/nghĩ không gọi API scene của Unity**: `transform`, `GameObject`, `Instantiate`/`Destroy`, `Find*`, `Physics*`, `Renderer`,
   `UnityEngine.Random` (chỉ chạy luồng chính — dùng `System.Random`/`DetHash`, mà A4 vốn đã đòi), `UnityEngine.Time` (dùng `SimTime`,
   luật M23.1). Kiểu giá trị `Vector2/3`, `Mathf` dùng thoải mái. "Code tick" = hàm đăng ký `GameClock` (`Subscribe`/`OnSlowTick`/…), `JobGiver`, `*Ops`, bộ não NPC/quái.
3. **Một chiều:** sim → dữ liệu → lớp vẽ chép theo. Lớp vẽ không ghi ngược vào sim; input của người chơi đi vào sim dưới dạng **lệnh**
   (`MoveTo`, `ForceJob`…), không phải sửa `Transform` rồi để sim đọc lại.
4. **Việc nặng tách 2 pha** khi phải chạy nền: pha 1 (luồng chính) làm hết phần GHI state dùng chung; pha 2 (luồng nền) **thuần đọc
   mảng**, không chạm API Unity, không ghi `Dictionary`/`List` dùng chung, không bắn event; kết quả về qua `ConcurrentQueue` và chỉ được
   giao nếu **vẫn là yêu cầu mới nhất** (id yêu cầu / thế hệ quần thể / đích chưa đổi). Trong lúc chờ, đối tượng giữ nguyên hoặc đứng yên —
   không bao giờ dịch chuyển. Mẫu: `RegionPathfinder.PrepareQuery`/`ExecutePreparedQuery`, `BeastBehaviour.RequestPath` (7da03a8e),
   `CityNpcSystem.BeginIndoorWalk` (77ca34fb).
5. **Sim không gọi thẳng vào UI.** Người nghe EventBus thường là MonoBehaviour/UI — sim chạy luồng khác thì họ bị gọi từ luồng nền và
   hỏng. Code mới chọn một trong hai: (a) UI tự **đọc** dữ liệu sim khi vẽ (kéo, không để sim đẩy); (b) sự kiện sim → UI đi qua **hàng
   đợi**, phát trên luồng chính ở điểm đồng bộ (cùng ý CLAUDE.md §6.1 luật 3). Sự kiện sim → sim vẫn gọi thẳng được. Hôm nay mọi thứ còn
   chung luồng chính nên gọi thẳng chưa hỏng — luật này để không đẻ thêm chỗ phải gỡ.
6. **Ngoại lệ có chủ đích + NỢ có tên:** di chuyển vẫn chạy mỗi khung trong `Update` với `SimTime.DeltaTime` (doc `TickRate`) — được, miễn
   vị trí ghi vào dữ liệu (mục 1). Nhưng đây là phần sim chạy nhiều nhất: muốn có luồng sim riêng thì phải dời di chuyển vào tick cố định
   rồi lớp vẽ **nội suy** giữa hai tick (thân 3D đã nội suy, xem doc gốc vẽ 3D) — đó là việc thật, không phải đổi chỗ. Composition root,
   `*Bootstrap`, renderer và UI đương nhiên được đụng Unity.

*Bằng chứng (2026-10-17, đo Profiler, map 5×5):* sim chỉ ~1,5 ms/khung nhưng có một cú khựng 25 ms nằm trong nhịp chậm của dân thành —
`BeginIndoorWalk` gọi `FindPath` đồng bộ (BFS tới-được ~10 ms + A* ~13 ms). Không đẩy nổi sang luồng khác nếu không tách pha, vì gần như
mọi hàm của tầng tìm đường **ghi cache khi được hỏi** (doc ở `RegionPathfinder`, "Tìm đường 2 PHA"). Tách xong: nhịp chậm max 7 ms.

*Phép thử:* hàm tick có chạy được trong test plain-C# **không cần scene** không (D2)? Không chạy được thì nó đang đụng Unity.

*Test canh:* `Run Sim Purity Tests` (`Tests/Editor/SimPurityTest.cs`) quét `*Ops.cs` + `JobGiver_*.cs` tìm `transform`/`GameObject`/`Find*`/
`UnityEngine.Random`/`Time.*`/`Physics`/`Instantiate`/`Destroy`. Lúc thêm luật có 24 chỗ cũ / 12 file; cùng ngày dọn còn **6** (lý do từng
chỗ ghi trong bảng mốc của test) — chốt **mốc theo số chỗ mỗi file**: vượt mốc = đỏ; dọn bớt thì test nhắc **hạ** mốc. Chỉ được hạ, không được
nâng. Số ngẫu nhiên không cần tất định trong code sim: `Core/SimRandom` (C# thuần, mỗi luồng một máy, cùng cách gọi với `UnityEngine.Random`).
Mục 5 (sim → UI) và code tick ngoài hai họ file này test KHÔNG bắt — vẫn dựa vào người viết.

---

## PHẦN C — BẪY HỆ THỐNG (đã dính, sẽ dính lại nếu quên)

### C1. "Code viết xong nhưng chưa gắn vào scene"
Lớp bug **im lặng tuyệt đối**: không log, không lỗi, chỉ là tính năng không tồn tại. Đã dính 2 lần
(`ForestLifecycleSystem`, `DayNightController`).

**Luật:** hệ MonoBehaviour tick-theo-thời-gian mà cần sống cùng map → `WildSceneBootstrap.EnsureMapSystems()`
tự `AddComponent`. Hạ tầng TOÀN CỤC → bootstrap ở `GameRoot`. **Đừng bắt từng scene nhớ.**

*Kiểm nhanh:* lấy `guid` trong `.cs.meta` rồi grep guid đó trong `Assets/Scenes/*.unity` — 0 kết quả = chưa
ai gắn.

### C2. Hai thư mục JSON song song KHÔNG tự đồng bộ
`Assets/Data/` (Editor đọc) ⟂ `Assets/StreamingAssets/Data/` (chỉ build thật).

File MỚI → tạo ở `Assets/Data` trước. Sửa file CŨ → **sửa cả hai**, `diff` xác nhận. Lỗi "file không tồn
tại" là `LogWarning`, nên **check console phải lọc cả warning**, không chỉ error.

### C3. Đo trước khi tối ưu
*Bug thật:* mất **4 vòng** sửa nhầm hệ vì suy luận từ code. Thủ phạm thật hoá ra là
`FindObjectsByType` trong vòng lặp (713ms → 3.26ms sau khi sửa).

**Luật:** mở Profiler trước, sửa sau. Suy luận từ đọc code không được tính là bằng chứng.

### C4. Bẫy PowerShell khi sweep code
- **`-replace` mặc định KHÔNG phân biệt hoa thường** → sweep `\(Disciple ` nuốt luôn biến `(disciple`.
  Đổi tên kiểu C# **phải dùng `-creplace`**.
- `\bDisciple\b` còn khớp **3 lớp ngoài kiểu**: segment namespace (`Sim.Disciples.Disciple`, thư mục
  `UI/Sim/Disciple`), tên **thành viên** (`.Disciple` property), và **field của event** (`e.DiscipleId`).
- File `.cs` đầy tiếng Việt → ghi bằng `[IO.File]::WriteAllText(path, text, UTF8-no-BOM)`. **Đừng
  `Set-Content`.**

### C5. Dời file trong Unity phải mang theo `.meta`
Cả `.cs.meta` **lẫn** `.meta` của thư mục (`Traits.meta` nằm cạnh thư mục `Traits/`). Quên = Unity cấp lại
GUID = đứt mọi tham chiếu.

### C6. MCP / Editor
- **Luôn check `is_playing` trước khi diễn giải state** — đã lỡ xoá UI runtime của phiên Play đang chạy vì
  tưởng là rác kẹt trong scene.
- Editor **mất focus thì `Update()` không chạy** (`Time.frameCount` đứng yên dù `realtimeSinceStartup` trôi)
  → `GameClock.CurrentTick` đứng yên dù không pause. Giải: gọi thẳng `GameClock.Advance(dt)` nhiều lần nhỏ
  (có `MaxTicksPerFrame` budget mỗi lần gọi, dư bị **vứt** chứ không tích luỹ).
- `Destroy()` ở Edit Mode **hoãn tới cuối frame**, mà `SaveScene` chạy trước đó → clone runtime bị **lưu
  vĩnh viễn vào scene**. Trước `SaveScene` phải `DestroyImmediate`.

### C7. Font không có emoji
`LiberationSans SDF` + 1 fallback: **không có emoji nào**, và cũng không có `−` (U+2212) hay `∞`.
Nhãn nút phải dùng **chữ**. Kiểm đúng cách:
`font.characterLookupTable.ContainsKey(cp) || fallback[i].characterLookupTable.ContainsKey(cp)`
(**đừng** dùng `HasCharacter(char)` — emoji là surrogate pair).

### C8. Backend save đúng ≠ scene có cách để Lưu
Cùng họ bug với C1, riêng cho save: `SaveManager`/`GameSaver` chạy đúng và verify Play Mode không có
nghĩa là **người chơi** lưu được — còn phải có nút/panel thật đặt trong ĐÚNG scene đang chơi.

*Bug thật (phát hiện 2026-08-27):* `SaveLoadPanel`/`AutoSave` chỉ được gắn ở `CityScene.unity`.
`WildLocal.unity` — scene chính, nơi gần như mọi hệ M19-M54 chạy — **không có nút Save/Load hay AutoSave
nào**. Bộ máy phía sau đúng 100%, nhưng người chơi ở Wild không có cách nào bấm Lưu.

**Luật:** scene gameplay mới (hoặc scene cũ chưa từng check) → grep `"SaveLoadPanel|AutoSave"` trong file
`.unity` đó **trước khi** báo "save hoạt động". 0 kết quả = scene đó không lưu được, bất kể backend đúng
đến đâu.

### C9. Bản Burst phải giống bản C# TỪNG BIT — và hai chỗ nó lệch mà không ai thấy
Áp cho mọi hệ đưa sang Jobs/Burst (DOTS hybrid, CLAUDE.md §6.1, luật 6: giữ bản C# + test so hai bản).

*Bug thật (2026-10-01, đưa A* NPC thành phố sang Burst):* hai bản ra đường KHÁC nhau dù logic chép y hệt.
Có 2 nguyên nhân độc lập, cả hai đều im lặng — đường vẫn hợp lệ, cùng độ dài, chỉ khác ô:
1. **Mono KHÔNG làm tròn kết quả trung gian về float32.** Đo: `g + e*D` lệch bản làm tròn chuẩn 2,4% số lần;
   cùng một biểu thức viết liền ⟂ tách qua biến tạm lệch nhau 20%. Burst thì chuẩn IEEE. Lệch ở bit cuối là đủ
   đổi thứ tự phá hoà ⇒ 26/310 truy vấn ra đường khác. `FloatMode.Deterministic` KHÔNG chữa được (lỗi ở phía
   Mono). **Sửa: ép `(float)(...)` từng bước ở CẢ HAI bản** — Mono tôn trọng phép ép (đo 0/2.000.000 lệch).
2. **`SortedSet.Add` BỎ khoá trùng, heap thì không.** Hai `g` khác nhau của cùng một ô làm tròn ra cùng một
   `f` ⇒ SortedSet giữ 1 mục, heap giữ 2, lấy ô ra 2 lần, tốn thêm lượt vào trần ⇒ đường rất dài chạm trần
   sớm hơn. Chỉ lộ trên **thành phố thật** (1/300 truy vấn) — lưới ngẫu nhiên nhỏ và cả bảng địa hình `null`
   đều KHÔNG bắt được (đã thử tắt phần sửa mà test vẫn xanh). Sửa: heap giữ `NativeHashSet` khoá đang nằm
   trong open.

**Luật:**
- Mọi phép float quyết định THỨ TỰ (so sánh, phá hoà, ngưỡng) phải ép `(float)` tường minh ở cả hai bản.
- Port cấu trúc dữ liệu thì port cả **ngữ nghĩa ngầm** của nó (Set bỏ trùng, Dictionary không thứ tự…), không chỉ API.
- Test đối chiếu phải chạy trên **dữ liệu thật** (data JSON thật, bản đồ thật cỡ thật), và phải được chứng
  minh là ĐỎ khi bỏ phần sửa — test chưa từng đỏ thì chưa chứng minh được gì.
- Burst trong Editor biên dịch **bất đồng bộ**: chạy test ngay sau khi sửa có thể còn chạy mã cũ. Xác nhận
  code mới đã nạp (reflection tìm member mới) trước khi tin kết quả.
- Thứ không port được thì nói thẳng: `Mathf.PerlinNoise` trong Burst ra kết quả KHÁC hẳn bản engine (đo ~50%
  điểm lệch, tới 0,049) ⇒ đưa worldgen sang Burst = thế giới đổi, phải tự viết noise và chỉnh lại tham số.
- Hàm Burst **gọi trực tiếp** (để chạy ngoài luồng chính) chỉ nhận struct BLITTABLE: field `bool` trong tham số ⇒ lỗi
  BC1063. Dùng `byte`.
- Làm tròn tường minh khiến bản C# chậm hẳn (đo ×5 ở bộ gen seamless — Mono không inline, `AggressiveInlining` không cứu).
  Giữ bản C# cho truy vấn LẺ, vòng lặp lớn phải đi bản Burst theo vùng — và soát mọi vòng lặp cũ còn gọi truy vấn lẻ
  (bị dính: `GenerateMap` lặp `TileAt` ⇒ test Large 69 s → 261 s; chuyển sang `FillRect` ⇒ 48 s).

---

## PHẦN D — QUY TRÌNH

### D1. "XONG" = người chơi chạm được
**Test Editor xanh KHÔNG phải là xong.** Câu hỏi phải trả lời được: *"người chơi chạm vào được chưa?"* —
và trả lời bằng **bằng chứng Play Mode**, không phải bằng số test.

Trạng thái phải ghi rõ một trong ba: `ĐÃ CODE` / `TEST XANH` / `ĐÃ VERIFY PLAY MODE`.

### D2. Mỗi hệ mới có test plain-C#
`Tests/Editor/Mx*Test.cs`, chạy qua menu `Nizhangjie/Run Mx Tests`, **không cần Play Mode**. Test viết bằng
C# thuần để chạy được trong Editor — đó là lý do sim tách khỏi MonoBehaviour ngay từ đầu.

Ngoài test chức năng, hệ nào có **ranh giới kiến trúc** thì phải có thêm **test canh ranh giới** (xem B8).

### D3. Grep trước khi sửa hay xoá
Dự án có file legacy trùng tên (bảng đầy đủ ở `CLAUDE.md §10`). Trước khi sửa/xoá: grep xem ai `new` nó,
ai inject nó. **Nghi ngờ thì hỏi.**

### D4. Ngôn ngữ
Comment/doc **tiếng Việt**, identifier **tiếng Anh**. Verb-action hiện thẳng lên nút (`Chop`/`Mine`/`Haul`…)
dùng **tiếng Anh**; nội dung/lore vẫn tiếng Việt.

### D5. Sửa nguyên nhân, không sửa triệu chứng
*Ví dụ:* bug "kiếm nằm ở ô mũ" — nguyên nhân thật là **bảng số slot SAI trong tài liệu** `M2_README.md`,
không phải data lỗi. Sửa data mà không sửa tài liệu thì bug sẽ mọc lại từ chính tài liệu đó.

Tương tự: test đỏ thì phải xác định **bug CODE hay bug FIXTURE** rồi sửa đúng chỗ — **đừng sửa test cho
xanh** nếu code mới là thứ sai.

### D6. ~~Đừng nhồi field chết~~ — ĐÃ BỎ (2026-10-12, user chốt)
Luật cũ: "chưa có case thật thì chưa thêm field/handler". **User bỏ luật này** — được dựng trước cơ chế/field
khi thấy rõ nó sẽ cần (vd điểm neo phóng vật bay theo cơ thể trước khi có skill phun từ miệng). Các chỗ trong
code/doc còn ghi "luật D6"/"chờ case thật" là lý do LỊCH SỬ của quyết định lúc đó, không còn là luật cấm.
Vẫn giữ tinh thần còn lại: field mới phải có **mặc định hợp lý** để data cũ/implementer không phải bịa giá trị.

---

## PHẦN E — CHECKLIST TRƯỚC KHI NÓI "XONG"

- [ ] Biên dịch sạch **0 lỗi** (và check cả **warning** nếu đụng JSON)
- [ ] Test của hệ mới xanh
- [ ] Test hồi quy của các hệ bị chạm xanh
- [ ] Nếu đụng ranh giới kiến trúc: test canh ranh giới xanh
- [ ] Code tick mới không đụng `Transform`/`GameObject`/API scene; trạng thái sim nằm trên dữ liệu, lớp vẽ chép theo (B10)
- [ ] **Verify Play Mode** — nêu được cụ thể đã bấm/chạy gì và thấy gì
- [ ] Trạng thái mới cần sống qua save/load? → đã có trong `SaveData` + `Capture`/`Restore` ở cả 3 chỗ
      trong `SaveManager` (B9), và scene người chơi thật sự chơi có nút Lưu (C8) chưa?
- [ ] JSON sửa ở **cả hai** thư mục
- [ ] `CLAUDE.md` cập nhật: cơ chế, **lý do**, bẫy, trạng thái verify
- [ ] Nợ/giới hạn **nói thẳng**, không giấu
