---
name: project_modding_strategy
description: "Chiến lược modding (KHÔNG phải milestone) — modding = kiến trúc nội bộ tình cờ mod được, KHÔNG phải nền tảng được hỗ trợ; không Lua, không hợp đồng hook ổn định, không on-call; chủ động giữ seam sạch, khi cộng đồng mạnh thì bless loader họ tự dựng"
metadata:
  node_type: memory
  type: project
  originSessionId: e5d4a8eb-636c-42c7-90b9-067c83108e87
  modified: 2026-09-07T06:12:48.950Z
---

**Quyết định chốt 2026-09-07 (bàn dài trong session).** Bối cảnh: user là solo dev, còn 1 dự án khoa
học robotics song song → **không muốn trực ban bảo trì hệ modding hàng ngày**. Đây là ghi chú chiến lược,
không phải milestone. Nền kỹ thuật đã có ở [[project_data_registry]].

## Nguyên tắc gốc — TÁCH 2 TẦNG
"Không doc / không hứa / không on-call" áp cho **tầng hook C#**, KHÔNG áp cho **tầng data-driven**.

- **Tầng data (JSON def, cấu trúc folder, `namespace:id`, schema các Def):** ship KÈM game ở
  `StreamingAssets/Data`, đọc được trần trần. Nó **tự lộ cấu trúc — và đó là CHỦ Ý**, là phần trả công
  cho bao nhiêu công sức xây hệ data-driven. Modder học bằng cách đọc file ship sẵn; schema tự nó là tài
  liệu. Đây là bề mặt moddable "miễn phí", luôn mở, và gánh bảo trì ~0 (thêm data không đụng code). KHÔNG
  giấu, KHÔNG obfuscate cái này.
- **Tầng hook C# (`IXContributor`/registry/interface):** advanced, at-your-own-risk. CHỖ NÀY mới là
  "không công bố API, không viết doc, không hứa ổn định ⇒ không nghĩa vụ bảo trì ⇒ không on-call".

Modding thành *việc thật* (đầu tư mạnh) chỉ vào ngày user chủ động quyết coi modder là đối tượng phải
phục vụ — mà lúc đó cộng đồng mạnh cũng đủ sức tự lo (xem mục "ngã ba" cuối).

## Xếp hạng gánh bảo trì (lý do loại Lua)
- JSON def + `namespace:id` + validate pass + topo-sort: **~0** (thêm data không đụng code, tự sống).
- `ConditionExpr` DSL nhỏ đóng (~10 toán tử): **rất thấp** (nhỏ + đóng ⇒ không churn).
- Hook/registry C# **không hứa gì**: **thấp** (ship chỗ tiện, mod gãy khi refactor = việc của modder).
- Harmony + hợp đồng hook ổn định: **cao**.
- **Lua / embedded scripting VM: CAO NHẤT — ĐÃ LOẠI.** Binding surface = mọi type/method phơi ra, lớn
  thêm *mỗi lần* thêm hệ; engine này lớn nhanh (66+ mốc) ⇒ thuế liên tục. Cũng đụng luật determinism
  (pairs order, rng, os.time) + serialize Lua state khi save = địa ngục.

## Việc CHỦ ĐỘNG nên làm (rẻ, không on-call)
- Giữ seam mạch lạc: registry nhất quán, kỷ luật `namespace:id` (add vs override tường minh), validate
  pass fail-fast có provenance, `ModLoader` topo-sort, cô lập per-pack try/catch.
- Ship hook ở các đường nối tự nhiên **khi xây từng hệ** (đang làm — mỗi hệ 1 `XHandlerRegistry`).
- Append-only rẻ, làm sớm khỏi phá sau: thêm `priority` int vào interface hook base; mỗi registry khai
  rõ ngữ nghĩa **pick-one vs run-all** + log khi mod thứ 2 giành cùng key (pick-one).
- (Tuỳ chọn, trả nợ cho CHÍNH user) xây `ConditionExpr` / cây predicate + effect-chain JSON **một lần**:
  gom các mini-DSL điều kiện đang tự chế rải rác (`EvolutionPathDef.conditions[]`, `IncidentDefinition`
  điều kiện, `StorytellerSystem` ngưỡng). Lợi: sửa logic quest/event/evolution không cần Unity recompile
  (nhớ cái reimport 10+ phút ở [[project_m65_shop_framework]]).

## Việc KHÔNG làm
- KHÔNG thêm Lua/scripting VM nhúng.
- KHÔNG chính thức hỗ trợ Harmony, KHÔNG ký hợp đồng hook ổn định.
- KHÔNG viết doc modding / công bố API.

## Harmony ≠ đường `.dll` hiện tại
Đường `IXContributor` scan (`GameRoot` mục 6i) = modder implement interface, engine gọi ở hook đã định
= **đăng ký VÀO điểm mở rộng**, không monkey-patch. Đó là mô hình an toàn (giống event-subscription của
Factorio; Factorio nuốt hàng trăm mod nhờ CẤM monkey-patch + load order tường minh, không nhờ Lua).
Harmony chỉ là **cửa thoát hiểm** cho chỗ quên chừa hook — modder tự chịu. Mỗi lần có người phải dùng
Harmony = tín hiệu thiếu hook chỗ đó, nhưng **gom lô theo nhịp release**, KHÔNG hotfix real-time.

## Mô hình Minecraft (đường thoát cuối, đã chấp nhận)
"Ship gì đâu, hứa gì đâu" vẫn ra hệ sinh thái mod khổng lồ — cộng đồng tự dựng Forge/Fabric trên bytecode
obfuscate. Vị thế user còn hơn Mojang thời đó: modder khởi đầu từ JSON def + registry sạch, không phải
bytecode. Thứ DUY NHẤT user "nợ" để đường này mở: **đừng chủ động chống lại** — không obfuscate nặng,
không anti-tamper, giữ assembly load được, để lộ 1 entry point tàm tạm.

## Khi cộng đồng mạnh — ngã ba
Cộng đồng mạnh ⇒ họ **tự đủ sức dựng modding API**, nhất là khi được cấp framework tử tế sẵn. Lúc đó
KHÔNG tự build API từ đầu — **bless cái loader họ đã dựng** (thêm vài hook chính thức mirror pattern cộng
đồng, có khi mời chính người viết loader). Nhẹ hơn nhiều + thiện chí cao hơn. Đúng đường Mojang cuối cùng
đi. Chất lượng seam CHỦ ĐỘNG bây giờ quyết định cộng đồng hội tụ về *một* loader hay đẻ 3 loader không
tương thích cãi nhau (Forge vs Fabric vs Rift).

Liên quan: [[project_data_registry]] (nền kỹ thuật: GameDatabase/Def/namespace:id/ModLoader), 
[[feedback_core_sim_before_xianxia]] (engine tổng quát, xianxia chỉ 1 pack), [[project_m64_combat_mode_framework]] 
(§12 "modding lối đánh" 3 tầng — ví dụ mẫu của pattern registry+contributor).
