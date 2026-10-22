---
name: project-m48-chronicle
description: "M48 — sử ký khách quan có trần (MemoryEventDef + WorldChronicle), đợt 1/3 của hệ trí nhớ hướng tới hội thoại RNG-theo-người-nghe/người-nói/môi trường"
metadata: 
  node_type: memory
  type: project
  originSessionId: 38f6f05f-a15c-443e-a952-6a5a0dacc2bf
  modified: 2026-08-21T01:35:15.659Z
---

**M48 (2026-08-21)** — nền cho hệ trí nhớ. Mục tiêu cuối user chốt: hội thoại nói được thứ **vừa RNG vừa
phụ thuộc người nghe ⟂ người nói ⟂ môi trường**, kiểu lai RimWorld + Dwarf Fortress + Wildermyth. Đợt này
**cố ý chưa đụng hội thoại** (user: *"MemoryEventDef + sử ký có trần trước, sau đó test rồi đưa vào hội
thoại sau"*).

## Thiết kế 3 tầng (mới xong tầng 2)

1. `GlobalEventBus` — bắn tin rồi thôi, không lưu.
2. **`Core/Chronicle/WorldChronicle`** — sự thật KHÁCH QUAN, cuốn chiếu, trần cứng. ✅
3. Ký ức TỪNG pawn (ai TIN gì — sai được, nghe kể lại được) — **chưa làm**.

**Lý do tầng 2 không phải bản nháp của tầng 3** (câu chốt của cả thiết kế): **nhân chứng muộn**. Ba ngày
sau có người đi ngang thấy cái xác rồi hình thành niềm tin về chuyện đã xảy ra. Không giữ bản ghi gốc thì
nhân chứng chỉ sinh ra được ĐÚNG lúc sự việc diễn ra.

## Phân biệt sống-còn: BẢNG ⟂ BẢN GHI

User nói "bảng đó dài chừng nào cũng khả thi" — **đúng với `MemoryEventDef`** (500 loại = cùng chi phí
runtime, modder thêm 1 file JSON), **SAI với `ChronicleEntry`**: bản ghi phình theo **THỜI GIAN CHƠI** chứ
không theo nội dung ⇒ bắt buộc có trần. Nhầm hai thứ này = "chơi 3 tiếng mượt, 30 tiếng save phình".

## Quyết định đáng nhớ

- **HAI TRẦN khác bản chất, giữ cả hai**: `maxEntries` = trần BỘ NHỚ (cứng, cắt từ cũ nhất, **không nể
  weight** — trần có ngoại lệ thì không còn là trần); `retentionDays × Weight` = trần THỜI GIAN (mềm).
  Chỉ 1 trong 2 đều thủng: chỉ-thời-gian → 1 đêm hỗn chiến thổi sổ vô hạn; chỉ-số-lượng → thế giới yên ả
  giữ mãi chuyện vặt năm ngoái.
- **Khoá actor/target = `string` theo ĐÚNG quy ước `HealthRegistry`** (`IWildCombatant.HealthId`) — cố ý,
  để tầng ký ức sau ghép được "vết thương này" với "trận đánh kia". KHÔNG dùng `PawnRef` (lý do chép
  nguyên của HealthRegistry: `PawnTier` là enum append-only đã serialize).
- **`RequiresTarget` SUY từ template có `{target}`** — không khai field thứ hai (khai 2 chỗ là để lệch).
- **GHI nghiêm ngặt ⟂ ĐỌC khoan dung** (bất đối xứng có chủ ý): `Record` từ chối def lạ/thiếu actor
  (luật B7 — bản ghi không kể lại được chỉ tổ chiếm trần); `RenderText` gặp def bị gỡ vẫn hiện id thô.
- **Dọn hạn `RemoveAll` chứ không cắt từ đầu** (hạn khác nhau theo weight), nhưng **chỉ quét khi đồng hồ
  nhích đủ 1 giờ in-game** — thiếu cửa này thì 500 bản ghi cùng 1 tick = quét 500 lần cả sổ (O(n²)).
- **Tick TƯƠNG LAI không bao giờ hết hạn** — đồng hồ lùi (save/load, test) mà dọn nhầm là mất data vĩnh viễn.

## ⚠ 2 bẫy đã dính THẬT trong đợt này

1. **Class trùng tên segment namespace** — đặt `class Chronicle` trong `namespace Core.Chronicle` gây
   `CS0118: 'Chronicle' is a namespace but is used like a type`. Đây là lần thứ BA dự án dính họ bẫy này
   (`Time` ở CLAUDE.md §9, `Cell` ở [[project_cellindices]]). **Đổi thành `WorldChronicle`.** Luật rút ra:
   đừng bao giờ đặt class trùng tên segment cuối của namespace chứa nó.
2. **Cửa sổ debug suýt làm đỏ chốt kiến trúc** — `ChronicleWindow` cần `DiscipleRegistry` (Sim) để đổi
   id→tên, mà `Arch_CoreNeverDependsOnSim` quét **TOÀN BỘ `Scripts/Core/`**. Đặt ở
   **`UI/Editor/ChronicleWindow.cs`** chứ không phải `Core/Chronicle/Editor/`.

## Cửa sổ debug là BẮT BUỘC, không phải tiện nghi

Sử ký là hệ đầu tiên mà "chạy đúng" và "chạy sai" **trông giống hệt nhau từ ngoài** — không hiện gì, không
lỗi gì (đúng lớp bug im lặng luật C1). Không nhìn được sổ thì không ai biết hệ nào QUÊN ghi hay ghi THỪA,
và cả hai chỉ lộ ra rất lâu sau ở tầng hội thoại xây bên trên. Menu `Nizhangjie → Chronicle (Sử Ký)`.
Cửa sổ **cố ý KHÔNG gọi `Resolve()`** khi sổ vắng mặt — làm vậy tự tạo sổ rỗng trông như thật, che đúng
lỗi cần thấy.

## Nợ đã biết, nói thẳng

**Sử ký CHƯA đi vào save** — load lại là sổ rỗng. Chấp nhận được vì chưa có gì phụ thuộc, nhưng **tầng ký
ức/nhân chứng muộn đợt sau SẼ cần** → phải thêm `SaveSystem/ChronicleIO` (mirror `WorldEventIO`).
Ghi ra theo luật B5.

## Trạng thái

**ĐÃ VERIFY PLAY MODE 2026-08-21** (WildLocal, pawn thật, qua ĐÚNG 3 đường production —
`WildCombatOps.ResolveAttack` / `SocialInteractionOps.Apply` / `DiscipleDiedEvent`): sổ 0→1→2→3, câu kể
*"Đệ Tử Vàng đánh Đệ Tử Đen (17.95 sát thương)"*, hạn đúng tỉ lệ weight **10.5 / 30 / 120 ngày**,
**5000 bản ghi = 3 ms**, trần giữ đúng 50. Test `Run Chronicle Tests` 49/49; hồi quy Decoupling 42/42
(gồm arch guard), WildCombat 53/53, Social 22/22, Grief 28/28.

Liên quan: [[project_m41_pawn_roster]], [[project_m29b_social]], [[project_m35_grief]],
[[project_dialogue_vn_panel]], [[project_code_system_rule]].
