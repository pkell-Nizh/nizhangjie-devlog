# Case study 12 — Trồng trọt: thiết kế tổng quát trước, rồi 6 bước = 6 commit (23/10/2026)

> **Tóm lược 3 dòng**
> - **Nhầm mục đích:** Định dựng một hệ trồng trọt mới.
> - **Thiết kế đúng ra sao:** Thiết kế tổng quát trước, rồi mở rộng khung có sẵn (khối có nấc, random tick, registry) theo 6 bước.
> - **Bài học:** Mỗi bước một commit có test; thiết kế bằng văn bản trước khi code.
>
> *Các con số bên dưới lấy từ ghi chú đo lúc đó, không phải thứ cần thuộc lòng.*

**Kỹ năng:** thiết kế hệ thống data-driven, mở rộng khung có sẵn thay vì viết hệ mới, chia việc thành bước kiểm chứng được.

## Yêu cầu
"Trồng trọt — cần thiết kế trước để tổng quát hoá." Tài liệu thiết kế: [`docs/FARMING_DESIGN.md`](../docs/FARMING_DESIGN.md) (viết *trước* khi code).

## Nguyên tắc thiết kế (và vì sao đúng)
- **Không dựng hệ mới.** Mở rộng khung có sẵn: khối có nấc (stage) + `BlockTicks` random tick + handler registry.
- **Cây = khối có object con `plant` (`PlantSpec`);** loader dựng spec từ field cũ (`growDays`/`plantOn`/`harvest*`) → *file cũ không phải sửa*.
- **Điều kiện mọc = danh sách factor** (`IGrowthFactor` registry): light, sky, soil, temperature, water_adjacent, submerged, crowding, biome. Pack thêm factor mới (vd. `xianxia:lingqi`); *factor lạ = 1* (không làm hỏng).
- **Kết cục nấc cuối = `onMature` registry** (stay / tree / grow_up / fruit_beside / spread).
- **Đất = khối có `soil`** (tag + độ phì + độ ẩm); cây khai *tag* thay vì id; cuốc `core:till`, phân bón `core:fertilize`.
- **AI:** `JobGiver_GrowPlant` làm trọn vòng (hái → gieo → cuốc); gỡ dữ liệu `feat_crop_*` trùng từ thời 2D.

## Bốn câu hỏi tôi chốt (§11)
1. Chunk ngoài tầm: bù tăng trưởng khi nạp lại, **trần 10 ngày**.
2. Cuốc hay không do từng loài tự khai (tag đất).
3. Lõi chỉ dừng lớn; **héo chết để pack tự khai** (giữ lõi trung tính).
4. Mưa tưới ruộng: thời tiết khai cờ `wetsSoil`.

## Sáu bước = sáu commit (đều ngày 23/10)
| Bước | Commit | Nội dung | Test |
|---|---|---|---|
| 1 | `125c67fd` | điều kiện mọc (`PlantSpec`, `PlantGrowth`) | 25/25 |
| 2 | `43833fe6` | đất ruộng, cuốc gỗ/đá, độ ẩm, mưa tưới | 42/42 |
| 3 | `f483eb78` | bảng rơi khi hái, cây một vụ, liềm, lúa mì/cà rốt/khoai | 55/55 |
| 4 | `c68deb72` | mía, bí, nấm, cỏ cao (nguồn hạt), kết cục grow_up/fruit_beside/spread | 69/69 |
| 5 | `b5c86b23` | pawn làm ruộng trọn vòng, chọn hạt theo vùng | JobGiverGrowPlant 22 |
| 6 | `2fa84d71` | bù tăng trưởng vùng ngoài tầm + phân bón | — |
| 7 | — | pack xianxia — **hoãn theo yêu cầu** | — |

## Bẫy ghi lại cho lần sau
- `BlockDefinition` đọc bằng `JsonUtility` nên object con `plant` *không bao giờ null* → hỏi "có phải cây" bằng `def.Plant`, đừng `plant != null`.
- Chạy test ngay sau `refresh_unity` có thể còn code *cũ* mà data đã *mới* (domain reload chưa xong) → fail giả; gọi `refresh_unity` lần nữa với `wait_for_ready`.
- `OnBlockReplaced` từng bật mọi cây phía trên khi khối dưới thành không đặc (kể cả chỉ đổi nấc) → nay hỏi `SupportOk`.
- Hỏi "đã chín chưa" qua `IsRipe(stage)`, đừng đọc `harvestFromStage`.
- `Zone.SelectedCropId` giờ là id *hạt*; save cũ `crop_x` đọc thành `x_seed` (tương thích ngược).

## Bằng chứng
[`docs/FARMING_DESIGN.md`](../docs/FARMING_DESIGN.md) · 6 commit liệt kê ở trên (repo game).
