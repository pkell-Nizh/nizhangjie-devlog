---
name: feedback-data-folder-split
description: "2 thư mục JSON song song Assets/Data (Editor) vs Assets/StreamingAssets/Data (build) — LUÔN sửa/tạo file mới ở Assets/Data trước, StreamingAssets là bản build"
metadata: 
  node_type: memory
  type: feedback
  originSessionId: e8341e22-894a-4ddd-9678-b55c3a10e7f3
  modified: 2026-07-29T18:19:57.624Z
---

Project có **2 bản JSON song song, không tự đồng bộ**:
- `Assets/Data/{folder}/*.json` — **Editor Play Mode đọc từ đây** (`JsonDataLoader.DataRoot` dùng
  `Application.dataPath` khi `UNITY_EDITOR`).
- `Assets/StreamingAssets/Data/{folder}/*.json` — chỉ dùng khi **build thật** (`Application.streamingAssetsPath`).

**Why:** 2026-07-29, tự tay tạo `daynight_config.json` (M24) và `needs_config.json` (M25) chỉ trong
`StreamingAssets/Data`, và sửa `depleteMode`/`restEffectiveness` cho các file JSON đã có sẵn cũng CHỈ sửa
bản `StreamingAssets`. Hậu quả: Editor Play Mode không đọc được file nào trong số đó — `JsonDataLoader`
log "File không tồn tại" (Debug.LogWarning, **không phải error** — dễ lọt qua nếu chỉ check
`read_console(types:["error"])`, phải check cả `"warning"`). Vì `Config.Load()` luôn có `Default()`
fallback trùng số với JSON mình viết, mọi thứ "nhìn như chạy đúng" khi user test (DayNight/Rest/Mood) —
nhưng thực chất chạy trên fallback, JSON thật chưa từng được đọc. `depleteMode` cho cây/quặng
(`local_features/*.json`) và `restEffectiveness` (camp_tent) cũng dính y hệt vì đó là file ĐÃ TỒN TẠI ở
cả 2 nơi — sửa 1 bản không đồng bộ sang bản kia.

**How to apply:**
- File JSON MỚI (config chưa từng tồn tại): tạo ở **`Assets/Data/{folder}/`** trước tiên (đây là bản
  Editor test). Có thể tạo thêm bản `StreamingAssets` cho build nhưng không bắt buộc lúc đang dev/test.
- SỬA file JSON đã tồn tại từ trước (content authored, vd `local_features/*.json`,
  `wild_buildings/*.json`): sửa **CẢ HAI** bản — dùng `diff` để xác nhận đồng bộ trước khi coi là xong.
- Sau khi sửa JSON, check console với `types:["error","warning"]` (không chỉ `"error"`) để bắt được
  cảnh báo "File không tồn tại" — nó là warning, không phải error.
- `Assets/StreamingAssets/Mods/` là NGOẠI LỆ — đó là chỗ ĐÚNG cho mod (kể cả lúc dev/test trong Editor,
  `ModLoader` luôn quét StreamingAssets/Mods bất kể Editor hay build). Chỉ riêng `Data/` (base game
  content) mới có 2 bản song song.
