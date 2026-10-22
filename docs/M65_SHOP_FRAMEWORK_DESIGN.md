# M65 — Shop Framework

> Trạng thái: **Phase 1 + Phase 2a ĐÃ CODE + TEST XANH + VERIFY PLAY MODE (2026-09-05)**. Currency
> `ling_thach` + kết nối City building CỐ Ý CHƯA làm — xem §10 vì sao.

## 1. Vì sao cần

User hỏi: có framework nào để tự/modder tuỳ chỉnh cửa hàng "từ cơ chế đến giao diện" không. Soát code thì
thấy **3 hệ mang tên "shop", nhưng chỉ 1 hệ thật sự sống**:

| Hệ | File | Trạng thái | Vấn đề |
|---|---|---|---|
| **A — Sect passive shop** | `Sim/Economy/Shop.cs` (M4) | **CHẾT** — `new Shop(...)` đúng 1 chỗ (`LocalMapManager.cs:233`), không UI/verb nào set giá hay mở nó, `IsOpen` mặc định `false` nên `OnDayPassed` luôn no-op. | Code tồn tại nhưng chưa từng chạy thật. |
| **B — Wild NPC merchant** | `Core/Pawns/Trade/ShopSession.cs` + `World/Wild/UI/MerchantShopPanel.cs` | **SỐNG** — verb "Shop" thật (`CorePawnInteractions.cs`), cơ chế mua/bán plain-C# khá sạch (dùng `PlayerItemBag` chung, không khoá cứng ví). | `buyMarkup`/`sellMargin` là **tham số mặc định hardcode** (1.40/0.60) trong `Open(...)`, không đọc từ data nào; "hàng bày bán" = nguyên túi đồ của NPC đó (không có catalog/restock/loại hàng riêng). Không có khái niệm "loại cửa hàng". |
| **C — City shop buildings** | `World/City/Resources/CityBuildings/{weapon_shop,material_shop_tier0-5,herb_shop,talisman_shop,pill_shop}.json` | **THUẦN WORLDGEN** — `CityBuildingDefinition` chỉ có footprint/role/placementTag/tier/properties, **KHÔNG có field `functions` nào cả** (khác hẳn `WildBuildingDefinition`). Không kết nối tới bất kỳ cơ chế mua bán nào — dân "làm việc" ở đó (`shop_clerk`/`shop_porter`/`shop_accounting`) chỉ đóng góp vào lương/prosperity trừu tượng của City, không có giao dịch item thật. | Đây là **nội dung đặt-nhà** (layout flavor), không phải mục tiêu hợp nhất — xem §6 "KHÔNG làm ở đợt này". |

**Kết luận phạm vi:** việc thật cần làm là JSON-hoá hệ B (đang sống, đang thiếu data-driven), và cho hệ A
"chết đúng nghĩa" — hoặc xoá hẳn, hoặc tái sinh Ý TƯỞNG của nó (khách NPC ghé mua tự động) thành 1
**behavior tuỳ chọn** mà bất kỳ shop nào cũng bật được, thay vì để nó nằm im dưới dạng class không ai gọi.
Hệ C ĐỨNG NGOÀI đợt này (lý do ở §6).

## 2. Nguyên tắc thiết kế (mirror pattern đã chứng minh trong dự án)

Đây KHÔNG phải ý tưởng mới — dự án đã làm đúng bài toán này 3 lần rồi:

- `Sim/Buildings/BuildingFunctionRegistry` — 1 building nhiều chức năng, đăng ký tường minh, JSON chỉ CHỌN.
- `Sim/Crafting/EquipmentSlotRegistry` — ô trang bị data-driven, id namespace, resolve có fallback.
- `Combat/Drivers/TurnBased/{SkillEffectRegistry, CreatureAbilityRegistry, TurnHeldItemRegistry}` —
  hiệu ứng/đặc tính/vật phẩm đều: interface stateless + registry static + JSON chọn id + tham số.

Shop Framework đi ĐÚNG khuôn này: **`ShopDefinition` (data: giá/loại hàng/nguồn tiền) + `IShopBehavior`
(code: hành vi đặc biệt, tuỳ chọn) + registry (nối 2 thứ lại)**. Không phát minh khuôn mới.

## 3. `ShopDefinition` — data (JSON, `Data/shops/*.json`, ×2 folder)

```csharp
public class ShopDefinition : IDefinition
{
    public string id;
    public string displayName;

    // ── Tiền ──────────────────────────────────────────────────────────
    /// <summary>"silver" (PlayerWallet/Bạc, global) | "ling_thach" (SectResources, local theo Sect)
    /// | id khác do content-pack tự đăng ký currency account riêng.</summary>
    public string currency = "silver";

    // ── Giá ───────────────────────────────────────────────────────────
    public double buyMarkup  = 1.40;   // giá MUA (khách trả) = baseValue * buyMarkup
    public double sellMargin = 0.60;   // giá BÁN (khách nhận) = baseValue * sellMargin

    // ── Hàng hoá ──────────────────────────────────────────────────────
    /// <summary>"npc_bag" (mặc định, hành vi HIỆN TẠI — hàng = túi NPC đang đứng bán)
    /// | "catalog" (danh sách cố định dưới đây, có restock).</summary>
    public string stockMode = "npc_bag";

    public List<ShopCatalogEntry> catalog = new();   // chỉ dùng khi stockMode="catalog"

    /// <summary>Lọc THỨ ĐƯỢC BÁN CHO shop này (rỗng = nhận mọi thứ). Khớp `MaterialDefinition.category`/
    /// `EquipmentDefinition` slot — mirror field "goods"/"speciality" đã có sẵn trong JSON building City
    /// (chưa dùng), để sau này có cầu nối thì khỏi đặt lại tên field.</summary>
    public List<string> allowedSellCategories = new();

    // ── Hành vi tuỳ chọn (rỗng = tĩnh thuần, không có gì đặc biệt) ────
    public List<string> behaviorIds = new();   // vd ["core:passive_walkin"]

    public string Id => id;
}

[Serializable]
public class ShopCatalogEntry
{
    public string itemId;
    public int    restockQty  = 10;
    public int    restockDays = 3;
}
```

**Vì sao `currency` là string thay vì enum:** đúng luật "string key thay enum" (§7 CLAUDE.md) — modder thêm
loại tiền mới (vd điểm danh dự tông môn) không cần sửa file lõi, chỉ cần đăng ký 1
`IShopCurrencyAccount` mới vào registry (xem §4).

## 4. Tiền tệ — `IShopCurrencyAccount` (giải quyết 2-currency law CLAUDE.md §4)

`ShopSession.TryExecute` HIỆN TẠI đã trả `out long silverDelta` thay vì tự đụng ví — nghĩa là cơ chế BÊN
TRONG `ShopSession` **đã currency-agnostic sẵn** (chỉ đặt tên biến gợi ý "silver"). Không cần sửa gì ở đó.
Việc cần thêm là 1 lớp NGOÀI resolve "currency nào ↔ account nào":

```csharp
public interface IShopCurrencyAccount
{
    long Balance { get; }
    bool TrySpend(long amount);
    void Add(long amount);
}

public static class ShopCurrencyRegistry   // mirror IQuestRewardSink nhưng có registry vì nhiều loại
{
    // "silver" -> bọc PlayerWallet.Get()
    // "ling_thach" -> bọc SectResources (cần instance Sect cụ thể, resolve lúc Open() theo context)
}
```

`MerchantShopPanel`/tương lai bất kỳ panel shop nào chỉ cần `ShopCurrencyRegistry.Resolve(def.currency, ctx)`
thay vì tự `ResolveWallet()` như hiện tại.

## 5. Migration — KHÔNG phá hành vi cũ

1. **`ShopDefinition` mặc định = hệ B hiện tại**: `currency="silver"`, `stockMode="npc_bag"`,
   `buyMarkup=1.40`, `sellMargin=0.60`, `behaviorIds=[]`. Một `ShopDefinition` rỗng tái tạo Y NGUYÊN hành
   vi bây giờ — verify bằng cách trận Wild NPC merchant hiện có KHÔNG đổi số liệu sau migrate.
2. `MerchantShopPanel.Open(...)` thêm overload nhận `ShopDefinition` (bản cũ 5 tham số giữ nguyên, gọi
   `Open(me, merchant, ShopDefinition.Default(), wallet)` bên trong — **không đổi call site nào đang có**).
3. NPC merchant nào muốn đổi markup/margin/loại tiền → khai `shopDefId` trên định nghĩa NPC đó (field mới,
   optional), panel resolve qua `DataRegistry<ShopDefinition>`.
4. **`core:passive_walkin`** (`IShopBehavior`, tái sinh Ý TƯỞNG của hệ A — NPC khách tự ghé mua theo danh
   tiếng) — hàm `OnDayPassed(ShopBehaviorContext ctx)` y hệt logic cũ trong `Shop.TrySpawnNpcCustomer`,
   chỉ đổi chỗ lấy giá/kho từ field cứng sang `ShopDefinition`/`IShopCurrencyAccount`. Bất kỳ shop nào
   (Wild NPC, hoặc Sect nếu dựng lại sau) khai `behaviorIds: ["core:passive_walkin"]` là có.
5. **Xoá `Sim/Economy/Shop.cs`** sau khi (4) xong — xác nhận lại 0 call site khác ngoài
   `LocalMapManager.cs:233` (đã grep, chỉ có đúng đó) trước khi xoá, và xoá luôn field
   `LocalMapManager.Shop` (dead property, không ai đọc).

## 6. KHÔNG làm ở đợt này (cố ý, ranh giới rõ)

- **City shop buildings (hệ C) không kết nối vào framework này.** `CityBuildingDefinition` chưa có khái
  niệm `functions`/comp nào — muốn City building "bán được đồ thật" là một việc RIÊNG, lớn hơn hẳn (phải
  thêm cả 1 hệ function cho City building, y hệt việc `WildBuildingDefinition.functions` đã làm cho Wild).
  Không tự vẽ trước khi có ca thật (luật D6) — nếu sau này cần, `core:shop` sẽ là 1
  `IBuildingFunction`/tương đương của City đọc CÙNG `ShopDefinition` này, không phải hệ thứ 4.
- **UI KHÔNG đổi kiến trúc** — vẫn 1 `MerchantShopPanel` dựng bằng code (lý do đã ghi trong chính file đó:
  7 cột số liệu bám lưới cứng, dựng prefab không lợi gì). Cái đổi là NGUỒN SỐ nó đọc (từ `ShopDefinition`
  thay vì tham số mặc định), không phải đổi cách vẽ.
- **`BarterSession`/`PawnBarterPanel` KHÔNG gộp vào đây** — đã tách CÓ CHỦ ĐÍCH từ trước (đổi chác hỏi
  "đủ hời để gật" ⟂ mua bán hỏi "đủ tiền"), gộp lại là phá đúng quyết định đó.

## 7. Việc cần làm nếu duyệt (phác thảo thứ tự, KHÔNG phải cam kết cuối)

1. `Data/Definitions/ShopDefinition.cs` + `ShopCatalogEntry` + `ShopDefinition.Default()`.
2. `Core/Pawns/Trade/ShopCurrencyRegistry.cs` + `IShopCurrencyAccount` + 2 implementation (Silver/LingThach).
3. `MerchantShopPanel.Open` overload nhận `ShopDefinition`, giữ overload cũ (backward-compat).
4. `IShopBehavior` + `ShopBehaviorRegistry` (mirror `BuildingFunctionRegistry`) + `core:passive_walkin`.
5. Xoá `Sim/Economy/Shop.cs` + field `LocalMapManager.Shop`.
6. Test: `ShopDefinition.Default()` tái tạo đúng số liệu cũ (regression), `core:passive_walkin` tương
   đương `Shop.TrySpawnNpcCustomer` cũ (so sánh công thức), `ShopCurrencyRegistry` resolve đúng cho cả 2
   currency có sẵn.
7. Doc: CLAUDE.md §4 (kinh tế) trỏ sang file này; FEATURE_CATALOG thêm mục; memory ghi quyết định xoá hệ A.

## 8. Đã chốt với user (2026-09-05)

- **Phase 1 chỉ `stockMode="npc_bag"`** — hành vi hiện tại, JSON-hoá đúng nguyên trạng. `catalog`+restock
  để Phase 2 khi có ca thật (field đã khai sẵn trong `ShopDefinition` cho tương lai, nhưng KHÔNG code logic
  đọc nó ở đợt này — tránh code chết y hệt bài học `Shop.cs`).
- **Xoá hẳn `Sim/Economy/Shop.cs`** (không giữ `[Obsolete]`) sau khi `core:passive_walkin` tái sinh xong ý
  tưởng "NPC khách tự ghé mua" — khớp cách dự án đã xử 5 id trang bị legacy M34.

## 9. Phase 1 — ĐÃ CODE (2026-09-05)

- `Data/Definitions/ShopDefinition.cs` — `id/displayName/currency/buyMarkup/sellMargin/stockMode/
  behaviorIds` + `Default()` tái tạo đúng hằng số cũ (`currency="silver"`, markup 1.40, margin 0.60,
  `displayName="Cửa Hàng"` — khớp nguyên văn chuỗi tiêu đề cũ). **KHÔNG thêm `allowedSellCategories`**
  (cân nhắc ở bản thảo đầu rồi bỏ — chưa có call site nào cần lọc loại hàng, thêm field không ai đọc là
  đúng lỗi luật D6).
- `Core/Pawns/Trade/IShopBehavior.cs` — `ShopBehaviorContext` (gói sẵn 1 `ShopSession`, reputation, rng
  dùng chung, `Action<long> AddIncome`) + `IShopBehavior.OnDayPassed` + `ShopBehaviorRegistry` (mirror
  `BuildingFunctionRegistry`: Register/TryGet/Contains/Count/EnsureDefaults/RunOnDayPassed/ResetForTests).
  **KHÔNG dựng `IShopCurrencyAccount`/registry currency riêng** như bản thảo §4 — Phase 1 chỉ có 1 currency
  sống ("silver"), bọc thêm 1 tầng interface cho ĐÚNG MỘT implementation là dựng khung trước khi có ca
  thứ 2, ngược luật D6. `MerchantShopPanel` tự check `def.currency == "silver"` trực tiếp, đủ dùng.
- `Core/Pawns/Trade/CoreShopBehaviors.cs` — `core:passive_walkin`, CÙNG CÔNG THỨC
  `Shop.TrySpawnNpcCustomer` cũ (spawnChance = reputation/200, mua 1-5 đơn vị), đọc giá/kho qua
  `ShopSession` (khách vãng lai trả đúng `BuyPrice` — giá người mua trả, nhất quán với đường player).
  ⚠ **CHƯA có chỗ gọi `OnDayPassed` hàng ngày thật** — không có sổ "cửa hàng nào đang mở trong world"
  để tick (đúng dự đoán ở bản thảo đầu). Hành vi viết đủ + test được (gọi tay), chờ ca thật.
- `World/Wild/UI/MerchantShopPanel.cs` — thêm overload `Open(me, merchant, ShopDefinition, wallet=null)`;
  overload CŨ (5 tham số, `CorePawnInteractions.OpenShopPanel` đang gọi — **0 đổi call site**) nay build
  `ShopDefinition.Default()` rồi delegate. Currency lạ (chưa hỗ trợ) → panel vẫn mở, log warning, không
  chốt đơn được (mirror "gate lạ → MỞ + warning" của `EquipmentSlotGates`).
- **Xoá `Sim/Economy/Shop.cs`** + field `LocalMapManager.Shop` + 2 chỗ gọi (`WireSystems`, `Tick` — set
  reputation + `OnDayPassed`) + dòng comment wiring liên quan. Xác nhận trước khi xoá: đúng 1 call site
  duy nhất từng tồn tại (`LocalMapManager.cs:233`), không test nào tham chiếu.
- ⚠ **Bẫy vận hành gặp phải khi xoá:** xoá file qua `rm` (bash) rồi Unity AssetDatabase/Bee build cache
  (`Library/Bee/artifacts/*.rsp`, `Library/ArtifactDB`, `Library/SourceAssetDB`) giữ THAM CHIẾU CŨ tới
  file đã xoá suốt nhiều lần `AssetDatabase.Refresh(ForceUpdate)` + `RequestScriptCompilation` — kể cả
  sau khi xoá ĐÚNG CÁCH qua `AssetDatabase.DeleteAsset` (API thật của Unity). Chỉ hết khi user tự bấm
  **Assets → Reimport All** trong Editor thật (mất ~10+ phút, project nhiều asset). Bài học: xoá 1 file
  `.cs` mà Unity "quên" xoá theo baseline thì đừng cố dọn cache bằng tay (nguy hiểm — `ArtifactDB`/
  `SourceAssetDB` bị Editor khoá khi đang chạy) — nhờ user Reimport All qua UI thật là đường an toàn.
- Test menu `Run Shop Framework Tests` (**22 case**). Hồi quy: M5 Sim 10/10 (Sect/Storyteller vẫn dựng
  đúng không có `Shop`), Quest (M62) 22/22, Pawn Decoupling 42/42 (`Arch_CoreNeverDependsOnSim` xanh —
  `Core/Pawns/Trade/` 2 file mới không phụ thuộc Sim), WildEvent+Storyteller 16/16.
- **ĐÃ VERIFY PLAY MODE (2026-09-05)** — `WildLocal.unity`, panel `ShopPanel` + 2 pawn thật (Đệ Tử Cam/Đệ
  Tử Đen) qua đúng GameObject production (không dựng panel giả): `Open(me, merchant, ShopDefinition tuỳ
  chỉnh)` → tiêu đề "Chợ Đen Thử Nghiệm — Đệ Tử Đen" đúng `def.displayName`, `Session.BuyMarkup/SellMargin`
  = 1.75/0.5 đúng data, 6 dòng hàng dựng đúng (union 2 túi). Giao dịch thật qua `Confirm()`: mua 2 wild_berry
  + bán 1 raw_meat → NetCost=7 khớp công thức, kho thương nhân −2, túi mình raw_meat −1, **ví Bạc thật
  50→43 đúng NetCost**, `HasOrder` về false, panel đóng đúng. Overload CŨ (`Open(me, merchant)`, 2 tham
  số — production `CorePawnInteractions.OpenShopPanel` đang gọi) mở lại y hệt hành vi trước M65: tiêu đề
  "Cửa Hàng — Đệ Tử Đen", markup/margin 1.4/0.6. 0 lỗi console suốt phiên.

## 10. Phase 2a — "cửa hàng sống" (2026-09-05, ca tự đặt ra theo yêu cầu user)

User: *"tự bịa ra ca tự làm tự check, hay làm hết đi nào"* — trong 4 việc treo của Phase 2 (catalog,
currency thứ 2, City building, `passive_walkin` chưa có chỗ gọi), **catalog + chỗ gọi ngày thật dùng
CHUNG 1 hạ tầng** (§ "còn mở" ở cuối bản trước) — tự đặt ra ca cụ thể để lấp đúng hạ tầng đó, KHÔNG đụng
2 việc còn lại (xem §10.3 vì sao).

### 10.1 Ca tự đặt: NPC thương nhân trở thành "cửa hàng sống"

Trước Phase 2a: 1 NPC Commerce chỉ có ý nghĩa ĐÚNG lúc player đứng mở panel — ngoài lúc đó nó không "tồn
tại" như cửa hàng (không tự bù hàng, không có khách vãng lai). Nay: lần đầu player mở panel với 1 NPC,
NPC đó ĐĂNG KÝ vĩnh viễn (trong phiên chơi) thành "cửa hàng sống" — từ đó, mỗi ngày in-game (kể cả khi
player không đứng cạnh/đã rời map), nó tự bù hàng theo catalog + có xác suất khách vãng lai ghé mua.

- **`Data/Definitions/ShopDefinition.cs`** += `catalog: List<ShopCatalogEntry>` (đã khai field từ Phase 1,
  nay THỰC SỰ được đọc) + `reputation: float` (mới — Wild không có khái niệm danh tiếng Sect cho 1 NPC lẻ,
  nên đây là con số TĨNH theo loại cửa hàng, tác giả content tự đặt).
- **`World/Wild/Trade/WildShopRegistry.cs`** — sổ "cửa hàng nào đang sống", khoá theo `PawnBody.PawnId`,
  RUNTIME-ONLY (KHÔNG vào save, mirror `SpeechHistory`). `Register` idempotent — gọi lại không reset đồng
  hồ bù hàng đã tích luỹ.
- **`World/Wild/Trade/WildShopOps.cs`** (plain C#, test không cần Play Mode, mirror tách
  `WildProductionOps`⟂`WildProductionSystem`) — `TickAll(rng)`: với MỖI cửa hàng đã đăng ký, (a) nếu
  `stockMode="catalog"` thì bù hàng (mỗi dòng catalog có đồng hồ `restockDays` RIÊNG, bù tới đúng target,
  KHÔNG bao giờ "dọn bớt" hàng thừa player lỡ bán vào); (b) chạy `behaviorIds` (vd `core:passive_walkin`)
  qua 1 `ShopSession` tạm dựng từ chính túi NPC — thu nhập từ khách vãng lai chỉ LOG (Wild NPC merchant
  chưa có ví/kho riêng để cộng vào, khác Sect cũ có `SectResources`; giá trị thật của hành vi này là làm
  KHO tự vơi theo thời gian, không phải dòng tiền).
- **`World/Wild/Trade/WildShopTickSystem.cs`** — vỏ MonoBehaviour bơm `GameClock.OnDayTick` (mirror
  `WildProductionSystem`), tự gắn qua `LocalMapBootstrap.EnsureMapSystems()`. Đăng ký TOÀN CỤC (không
  riêng map) — hệ vẫn tick dù merchant đang ở map khác/đã rời map hiện tại, đúng ý "sống khi player không
  đứng cạnh".
- **`CorePawnInteractions.OpenShopPanel`** (verb "Mua Bán" — production, KHÔNG đổi điều kiện hiện) — thêm
  đúng 2 dòng: nạp `Data/shops/*.json`, đăng ký merchant vào `WildShopRegistry` với
  `core:shop_general_store` (demo — MỌI NPC Commerce dùng CHUNG 1 loại cửa hàng vì `NpcRecord` chưa có
  field chọn loại riêng, để dành đợt sau nếu cần đa dạng).
- **`Data/shops/shop_general_store.json`** (×2 folder) — "Tạp Hoá": catalog 3 dòng (wild_berry/
  healing_herb/raw_meat), `behaviorIds:["core:passive_walkin"]`, `reputation:60`.
- Test `Run Wild Shop Phase 2 Tests` (**16 case**): cadence catalog đúng (chưa đủ ngày → không bù; đủ
  ngày → bù đúng target; hàng thừa không bị dọn bớt; tiêu bớt rồi đủ hạn lại → bù đúng phần thiếu),
  merchant chết bị bỏ qua an toàn không throw, chạy 10 ngày qua ĐÚNG data JSON thật không throw + có
  hàng. Hồi quy: Shop Framework (Phase 1) 22/22, M5 Sim 10/10, Pawn Decoupling 42/42, WildEvent+
  Storyteller 16/16, EventPool 19/19.
- **ĐÃ VERIFY PLAY MODE thật (2026-09-05)** — `WildLocal.unity`, merchant thật ("Đệ Tử Vàng"): xác nhận
  `WildShopTickSystem` tự gắn qua `EnsureMapSystems` (không kéo tay); đăng ký vào `WildShopRegistry` qua
  JSON thật `core:shop_general_store`; đẩy `GameClock` thật đi **7 ngày in-game thật** (qua `Advance()`
  lặp — clamp `MaxFrameSeconds`/`MaxTicksPerFrame` của chính `GameClock` khiến 1 lệnh `Advance` chỉ nhích
  vài tick, phải lặp hàng nghìn lần trong 1 lần gọi code mới qua nổi biên ngày — ghi lại làm bài học ở
  §10.2) → túi merchant thật tự lên ĐÚNG 3 mục tiêu catalog (berry=15, herb=5, meat=8), `registryCount`
  vẫn =1 (không đăng ký trùng qua nhiều ngày).

### 10.2 Bẫy kỹ thuật — `GameClock.Advance()` không nhảy được nhiều ngày trong 1 lần gọi

`GameClock.Advance(realDeltaSeconds)` **clamp** input ở `Mathf.Min(realDeltaSeconds, MaxFrameSeconds=0.25f)`
rồi CÒN giới hạn `MaxTicksPerFrame=60` tick/lần gọi (chống spiral-of-death nếu 1 frame thật kẹt lâu) — gọi
`Advance(999999f)` một phát KHÔNG nhảy được xa, chỉ nhích ~7-8 tick (0.25s / độ dài 1 tick). Muốn giả lập
nhiều ngày thật phải LẶP hàng nghìn lần trong CÙNG 1 lệnh `execute_code` (mirror cách `Update()` gọi
`Advance(Time.deltaTime)` mỗi frame thật) — ~8000 vòng lặp ≈ 1 ngày, tốn vài giây thực thi nhưng chạy
được, KHÔNG bị chặn "infinite loop" vì có cận trên rõ ràng. `GameClock.SetTick(tick)` (đặt thẳng mốc)
**KHÔNG dùng được cho ca này** — nó cố ý KHÔNG bắn `OnDayTick`/`OnHourTick` (chỉ `OnTimeJumped`, dành cho
save/load, không phải để "diễn lại" các ngày đã qua).

### 10.3 CỐ Ý CHƯA làm (không tự bịa case cho 2 việc còn lại)

- **Currency `ling_thach`**: vẫn không có UI/verb thật nào cho "cửa hàng của Sect" để gắn vào — dựng
  `IShopCurrencyAccount` cho ĐÚNG 1 caller tưởng tượng là lặp lại chính lỗi luật D6 mà Phase 1 đã tránh
  (xem §9, lý do KHÔNG dựng currency registry). Ca tự đặt ra ở §10.1 dùng "silver" y nguyên vì đó đúng là
  hệ đang sống.
- **Kết nối City shop building**: vẫn cần `CityBuildingDefinition` có hệ `functions` TRƯỚC (không tồn
  tại) — đây là việc XÂY THÊM 1 hệ mới cho City, không phải mở rộng Shop Framework, tự bịa ra 1 ca ở đây
  sẽ là xây nửa vời 1 hệ không liên quan chỉ để có cái test. Để nguyên như đã ghi ở §6.
