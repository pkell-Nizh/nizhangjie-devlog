# BÀN GIAO cho Antigravity — Nishangjie / Nizhangjie (Unity 6)

Chào Antigravity,

Mình là Claude, vừa làm với Loc (chủ dự án) đợt dựng nền tảng modding cho Nishangjie. Từ đây cậu sẽ đồng hành cùng Loc một chặng dài, nên mình viết lại gọn những gì đã làm, mấy cái bẫy mình đã cắn phải, và hướng đi tiếp — để cậu khỏi mò lại từ đầu. Dự án này tâm huyết, code sạch và có kỷ luật test; mong cậu giữ được mạch đó. Chúc cậu với Loc làm việc vui và ra được nhiều thứ hay. Có gì khó cứ đọc kỹ `CLAUDE.md` trước, nó cứu cậu nhiều lần đấy.

Thân,
Claude

---

> Tài liệu này = điểm khởi đầu cho phiên làm việc dài. Đọc 5 phút rồi vào việc.
> Ngày bàn giao: 2026-06-26. Người dùng: buihuuloc2006@gmail.com (comment/doc tiếng Việt, identifier tiếng Anh).

---

## 0. Bối cảnh 30 giây
- Engine: **Unity 6 LTS · URP 2D · C# 9 · namespace `Nizhangjie.*`**. Thư mục dự án thật: `Nishangjie/Nizhangjie/` (lồng nhau, dễ nhầm).
- Triết lý: **data-driven engine thuần modding** — logic tách khỏi data; mọi unit/quái/skill/vùng đất là JSON. "Tu Tiên" chỉ là nội dung mặc định (Official Mod 01).
- Đã xong tới ~M13 Phase 1. Vừa hoàn thành **nền tảng modding** (mục 1).

## 1. Vừa hoàn thành (2026-06-26) — nền tảng data-driven/modding ĐÃ TEST PASS trong Unity
1. **Newtonsoft.Json** (opt-in per-type qua attribute `[UseNewtonsoftJson]`) bên cạnh JsonUtility — mở khoá `Dictionary`/nested.
2. **`GameDatabase`** (`Data/Loaders/GameDatabase.cs`) = composition root: `LoadAll` (13 type) → `ResolveAndValidate` (resolve cross-ref + fail-fast validate) → `Freeze`. Gọi trong `GameBootstrap`.
3. **namespace:id** stamping (`core:beast_wolf`) cho def `: Def`; resolver có raw-fallback cho def cũ `: IDefinition`.
4. **ModLoader topological sort** (`Modding/ModLoader.cs` + `ModInfo.cs`): đọc `mod_info.json`, sắp xếp theo `dependencies` (Kahn), 1 mod hỏng không brick game.
5. **LocalizationManager** (`Localization/LocalizationManager.cs`): string-ID → text (`Data/lang/{code}.json`), merge base+mod, fallback. Đây là thứ tách "Linh Lực" vs "Mana".

**Test (menu Unity, không cần Play Mode):** `Nizhangjie/Run M Data Registry Tests` · `Run M ModLoader Tests` · `Run M Localization Tests`. Tất cả PASS.

## 2. NGUỒN SỰ THẬT — đọc trước khi code (đừng tin số chép tay)
- **`Assets/Scripts/CLAUDE.md`** — bản đồ + 4 luật kiến trúc + bẫy + file legacy. **Đọc đầu tiên.**
- **`Assets/Scripts/Nizhangjie/M_DATA_REGISTRY_DESIGN.md`** §9c — chi tiết đợt modding vừa làm.
- `M_COMBAT_DESIGN.md`, `M_MONSTER_DESIGN.md`, `M10/M11/M12/M13_*` — gameplay/worldgen/combat.
- Memory người dùng (Claude): `~/.claude/projects/.../memory/` có các note `project_*` tóm tắt quyết định.

## 3. Ba việc INFRA follow-up — KHÔNG cấp bách (game chạy bình thường nếu bỏ qua)
| Việc | Ưu tiên | Khi nào làm | Cách |
|---|---|---|---|
| Cắm Localization vào UI thật | TB | Khi đụng UI | Map `StatType`→`UI_STAT_*`; đổi `TMP_Text.text="..."` → `LocalizationManager.Get("UI_...")`; nghe event `LanguageChanged` để refresh |
| Gỡ hardcode `==` chuỗi content (combat) | Thấp | Khi đụng combat | `entityKind=="beast"`, `faction=="hostile"`, `case "bleed"` → đọc field Def thay `==`. Xem M_DATA_REGISTRY_DESIGN §1 |
| Mở rộng `GameDatabase.LoadAll` (jobs/city_buildings/region_archetype) | Thấp | Tùy | Thêm 1 dòng `LoadType<T>("folder")`; `JobDefinition` ở ns `Nizhangjie.LocalMap.Jobs` (cần using). Lazy-loader cũ đã lo |

> JSON Patch RFC6902: **HOÃN** — file-level override đang đủ cho v1.0/v1.3.

## 4. Việc GAMEPLAY làm được NGAY (giá trị thấy được — ưu tiên hơn mục 3)
1. **M13 Phase 2+** (`M13_*` doc) — hybrid colony-sim tu tiên. Phase 1 đã code → tiếp phase kế. **Giá trị cao nhất.**
2. **Author nội dung JSON** (giờ rất nhẹ): thêm beast/skill/item/faction/trait vào `Assets/Data/<folder>/`. Đăng ký tự động qua `GameDatabase.LoadAll`. Validate fail-fast lúc boot báo lỗi rõ.
3. **Combat redesign** (`M_COMBAT_DESIGN.md`) — RTwP + ATB gauge, đang dở.

## 5. Quy trình bắt buộc
- **Test plain-C# trong Editor** cho mỗi hệ thống: `Tests/Editor/Mx*.cs`, chạy menu `Nizhangjie/Run Mx Tests`. KHÔNG cần Play Mode. Thêm hệ thống → viết test.
- **Sync JSON CẢ 2 nơi**: `Assets/Data/` (Editor đọc) + `StreamingAssets/Data/` (build đọc).
- **Enum/TileType append-only** (đừng đổi giá trị cũ → vỡ save).
- Namespace khớp thư mục. Sim mới → wire vào tick sequence của `LocalMapManager`, đừng tick rời.

## 6. GOTCHAS sống còn (đã cắn ở phiên này)
- **Ghost-script (Unity)**: file `.cs` tạo từ ngoài lúc Unity đang refresh → có `.meta` nhưng KHÔNG vào compile → "type could not be found" dù code đúng, recompile vô ích. **Chẩn đoán:** mở `Library/Bee/artifacts/<dag>/Assembly-CSharp.rsp`, xem file có trong list không. **Fix:** xóa `.meta` của file + xóa `Library/ScriptAssemblies` & `Library/Bee` → mở lại Unity.
- **UTF-8 cho JSON tiếng Việt**: KHÔNG dùng PowerShell `Get-Content`/`Set-Content` (mangle BOM/diacritics). Dùng `[System.IO.File]` hoặc editor ghi UTF-8 **no BOM**. Verify: byte đầu = `7b` (`{`), không `EF BB BF`.
- **Newtonsoft**: type `[UseNewtonsoftJson]` PHẢI `[JsonIgnore]` mọi field `[NonSerialized]` runtime (Newtonsoft bỏ qua `[NonSerialized]` mặc định).
- **namespace stamping**: def `: Def` có Id = `"core:id"` (không phải raw). Đừng `DataRegistry<DefType>.Get("raw_id")` cho type `: Def` (sẽ miss). Hiện chỉ BeastSpecies/StatusEffect là `: Def`.
- **File legacy** (xem CLAUDE.md §10): combat stack cũ đã dời ra `_LegacyCombat/` ngoài Assets. Canonical combat = scene `combat_in_prototype` + `CombatCoreDriver`. Đừng noi theo file legacy.

## 7. Khuyến nghị thứ tự cho phiên dài
1. Verify lại 3 test menu mục 1 còn PASS (sanity).
2. Chọn 1 hướng gameplay mục 4 (gợi ý: M13 Phase 2 HOẶC author nội dung) — ra kết quả thấy được sớm.
3. Khi đụng UI/combat thì tiện tay làm mục 3 tương ứng (Localization/gỡ hardcode), đừng làm rời.
4. Mỗi thay đổi → viết/chạy test Editor + sync JSON 2 nơi + cập nhật doc `M_*` liên quan.
