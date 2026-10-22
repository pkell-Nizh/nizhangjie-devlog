---
name: project_editor_open_hang_2026-08-29
description: "Sự cố mở Unity treo + leo RAM 11GB ngày 2026-08-29, nguyên nhân + những gì đã đổi"
metadata: 
  node_type: memory
  type: project
  originSessionId: 8457b35e-2e48-451d-8770-4b0b33370d8c
  modified: 2026-08-29T07:34:04.657Z
---

**Triệu chứng (2026-08-29):** mở dự án kẹt mãi ở splash "Running managed callbacks: Initializing
InitializeOnLoad Types", RAM leo liên tục tới 9–11GB, không bao giờ mở xong. Editor.log dừng ngay sau
domain reload thứ 2 (cinemachine partial-class warnings) rồi "Shut down." (user tự kill).

**Nguyên nhân chính: cache `Library/` hỏng** do bị kill cứng nhiều lần giữa lúc asset import (còn
`ArtifactDB-lock`/`SourceAssetDB-lock`/`ilpp.pid`). Xoá sạch `Library/` + để Unity dựng lại → hết.

**Đồng phạm phụ đã xử luôn:**
- `Assets/Scripts/Editor/AilmentAssetGenerator.cs` — class `[InitializeOnLoad]` tự chạy
  `GenerateDefaultAilments()` qua `delayCall` mỗi lần nạp assembly. Các class `BleedDefinition`/… khai
  BÊN TRONG `BleedAilment.cs` (không phải file cùng tên) ⇒ Unity không tạo `MonoScript` ⇒ `CreateAsset`
  ghi asset `m_Script: {fileID: 0}` (hỏng) ⇒ lần sau `LoadAssetAtPath<T>` trả null ⇒ guard `== null`
  lọt ⇒ ghi đè lại → import → reload → lặp. Trái luật kiến trúc #1 (JSON-first). **Đã gỡ `[InitializeOnLoad]`
  + `delayCall`, giữ nút menu.** 4 asset hỏng trong `Assets/Data/Ailments/` đã xoá (0 code nào đọc chúng).
- `Packages/manifest.json` — **đã bỏ `com.coplaydev.unity-mcp`** (git package bám `#main`, session đó báo
  "UnityMCP ConnectionRefused"). Muốn dùng lại: `Window → Package Manager → + → Install from git URL`
  `https://github.com/CoplayDev/unity-mcp.git?path=/MCPForUnity#main`, ĐỪNG sửa tay manifest (dính bẫy BOM).

**Bẫy đã dính khi sửa:** `Set-Content -Encoding utf8` (PowerShell 5.1) ghi kèm BOM → Hub báo
"manifest.json is not valid JSON: Non-whitespace before {. Char: 65279". Phải ghi UTF-8 no-BOM (dùng
`[IO.File]::WriteAllText` hoặc tool Write). Xem [[feedback_powershell_utf8]].

**Kết quả:** mở nhanh, không leo RAM, playable. Còn 1 lỗi đỏ TMP `TextMeshProUGUI.Cull` NRE trong
`RectMask2D.PerformClipping` (bug Editor-side đã biết của uGUI/TMP Unity 6, vô hại; import lại
TMP Essential Resources nếu spam).
