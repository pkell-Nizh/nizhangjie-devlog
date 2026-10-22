---
name: project_fluids
description: "Bước 7 chất lỏng (2026-10-14) — 4 quyết định của user: chảy kiểu Minecraft, nước nông sâu 1 + hồ/biển sâu hơn, bơi + chết đuối + lội nước nông + dung nham bỏng, có xô; thứ tự 5 phần"
metadata:
  node_type: memory
  type: project
  originSessionId: d93f285a-5ead-48f8-9bcd-66fa9efac21c
  modified: 2026-10-14T12:16:02.162Z
---

User chốt 2026-10-14 (đừng hỏi lại):
- Chảy KIỂU MINECRAFT: khối nguồn + nước chảy loang có mức (nước ~7 ô, dung nham ngắn hơn), chảy xuống thành cột, 2 nguồn nước kề nhau đẻ nguồn mới (nước vô hạn).
- Độ sâu: nước nông + sông giữ sâu 1 khối; hồ + biển sâu thêm theo khoảng cách tới bờ (đổi bộ sinh thế giới — vùng nước đã có sẽ khác).
- Trong nước: avatar BƠI được; CÓ chết đuối (thanh dưỡng khí); NPC/quái LỘI được nước sâu 1 khối (chậm hơn), tránh nước sâu hơn; dung nham GÂY BỎNG, nước gặp dung nham thành đá.
- Có XÔ: múc / đổ một khối nguồn nước hoặc dung nham, có công thức làm xô.

Trước bước 7: nước/dung nham là khối ĐẶC thấp hơn bờ 1 khối, không chảy, không đập được, chắn sáng hẳn; mọi vùng nước sâu đúng 1 khối.

Thứ tự làm: (1) dữ liệu + luật chảy + test, (2) hồ/biển sâu hơn ở worldgen, (3) vẽ nước trong suốt + mặt nước theo mức, (4) bơi / chết đuối / bỏng / lội nước, (5) xô.

**Phần 1 XONG (2026-10-14, test `Run Fluid Tests` 14; Play Mode: 4027 ô nước của 9 ô world vẫn đúng loại ô, đào rãnh cạnh hồ ⇒ nước tràn, 3 ô thành nguồn mới, ô 4 mức 1, loại ô tự thành ShallowWater):**
- `World/Voxel/FluidFlow` (trong `BlockTicks.Fluids`, chạy theo lịch, trễ `fluidTickSeconds`, trần 3000 ô/khung). Trạng thái = nấc khối: 0 nguồn, 1..range đang chảy, range+1 đang rơi.
- `BlockDefinition.fluid/fluidRange/fluidTickSeconds/fluidInfinite/hardenWith/hardenSource/hardenFlowing`; `BlockIds.IsFluid/SameFluid`. `block_water` (range 7, 0,25 s, vô hạn, chắn sáng 2), `block_lava` (range 3, 1,5 s, gặp nước: nguồn ⇒ `block_obsidian`, đang chảy ⇒ `block_stone`). Cả hai `solid: false`, vẫn `unbreakable` (múc bằng xô ở phần 5).
- MẶT NƯỚC NGANG BỜ: `VoxelSolidSector.BaseHeight` không còn hạ ô nước 1 khối ⇒ khối nước nằm ở tầng mặt đất, đáy thấp hơn bờ 1 (lội lên xuống 1 bậc). Thế giới nhìn khác trước (sông không còn trũng).
- `VoxelWorld.DesiredType`: có chất lỏng phủ sàn ⇒ ô giữ loại nước gốc, hoặc thành `surfaceTile` của chất lỏng (nước ⇒ ShallowWater, dung nham ⇒ Lava). THIẾU luật này thì mọi hồ thành ô cát/đất đi được.
- `BlockPlacementOps.CanPlace` cho đặt khối vào ô chất lỏng.
- Vẽ tạm: `VoxelMapMesher.EmitFluid` — mặt trên hạ theo nấc, chỉ vẽ mặt giáp không khí; CHƯA trong suốt (phần 3).

**Phần 2 XONG:** `TerrainBlockConfig.Entry.maxDepth` (data `terrain_blocks.json`: chỉ `Water` = 4) ⇒ `VoxelSolidSector.SurfaceDepth` (BFS khoảng cách tới ô khác loại, ngoài hình coi như cùng loại) ⇒ lớp mặt dày nhiều tầng trong `BaseRuns`/`BaseBlockAt`. Play Mode: 1661 ô sâu ≥ 3. Nợ: hai ô world kề nhau tính độ sâu riêng ⇒ có thể lệch bậc dưới nước ở ranh.

**Phần 3 XONG:** `BlockDefinition.translucent` (nước) ⇒ `BlockIds.Translucent`; `VoxelMapMesher.EmitWater/WaterMeshes/RebuildWater` — lưới nước riêng mỗi chunk (cả trên trời lẫn trong hang), `VoxelGameView._matWater` (alpha 0,68, không ghi độ sâu, 2 mặt, renderQueue 3000), dựng lại cùng `OnChunkChanged`. Shader `VoxelView` thêm `_SrcBlend/_DstBlend/_ZWrite/_Alpha`. Dung nham vẫn đặc màu trong lưới thường. Ảnh `Assets/Screenshots/fluid_*.png`. Prototype cũ (VoxelViewPrototype/Seamless) không vẽ nước nữa — chấp nhận.

**Phần 4 XONG:** `KinematicBody.Submerged` + `BodyParams.Swim*` (≥ 0,5 ⇒ bơi: chìm chậm, giữ nhảy nổi lên, sát bờ giữ nhảy trèo ra, đứng đáy nhảy được). `WildPlayerToken.StepBody` đặt Submerged từ `VoxelWorld.FluidAt` (chân +0,4, đầu +1,6), đi chậm `swimSpeedMultiplier`, rơi xuống nước không đau; `UpdateFluidHazards`: dưỡng khí `airSeconds` 10 (`AirFraction`, `HeadSubmerged`), hết thì `drownDamageFraction` 10%/s; chất lỏng PHÁT SÁNG ở chân/đầu = bỏng `lavaDamageFraction` 10%/0,5 s. Số chỉnh ở `player_actions.json`. HUD POV: hàng bọt khí trên hàng no, chỉ hiện khi < đầy. Sông (`River`) nay `passable` 30% tốc như nước nông ⇒ NPC/quái lội qua; hồ sâu (`Water`) vẫn chặn — kể cả ô bờ sâu 1 (nợ nhỏ). Play Mode: ngập đầu ⇒ dưỡng khí 1→0 trong ~10 s rồi mất 12 máu/s (121 máu), đổi dung nham ⇒ mất 12 máu/0,5 s.

**Phần 5 XONG:** `World/Wild/Building/BucketOps` (Pick/Pour/FluidForBucket/IsBucket); `BlockDefinition.bucketItem` (nước ⇒ `water_bucket`, dung nham ⇒ `lava_bucket`); vật phẩm `bucket`/`water_bucket`/`lava_bucket`; công thức `recipe_bucket` 3 `iron_ingot` ở `building_smelter`. POV chuột phải (`VoxelGameView.UseBucket`): xô rỗng bắn tia TÍNH chất lỏng, chỉ múc ô nguồn; xô đầy đổ vào điểm đặt. Tia ngắm thường giờ XUYÊN chất lỏng (`AnyVoxel(fluids:false)`) ⇒ đập/đặt dưới nước được. Chưa bấm thử bằng chuột thật (luật đã test trên chunk thật).

**Sau bước 7 (2026-10-14, user: "nước giờ là thực thể vật lý, không cần tile gán mác nước"):** loại ô nước SUY TỪ KHỐI trong `VoxelWorld.DesiredType`: chất lỏng trong suốt trên sàn sâu 1 ⇒ `ShallowWater` (sông giữ `River`), sâu ≥ 2 ⇒ `Water`; chất lỏng khác ⇒ `surfaceTile` của nó (Lava); ô nước gốc mà cạn ⇒ theo khối đáy (mặc định Sand). `Apply` đồng bộ MỌI ô nước gốc của hình lúc dựng. Nhãn ô nước giờ chỉ còn là ĐẦU VÀO sinh thế giới (như biome của MC) + bản sao suy ra cho các hệ cũ đọc; nguồn sự thật là khối. Play Mode: 491/491 ô hồ sâu 1 thành nước nông đi được, 2062 ô sâu vẫn chặn.
- Độ sâu qua ranh ô world: `VoxelSource.FromWindow` chụp thêm lề `DepthMargin` = 8 ô loại ô GỐC (`MarginTypes`, dùng `_origType` để không tự nuôi lại loại ô đã suy) ⇒ `VoxelSolidSector.DepthField` tính trên vùng mở rộng. Test hồ hẹp vắt qua ranh ra đúng 3|2.
- Cát/sỏi rơi vào nước: `FallingBlocks.Land` đáp được vào ô chất lỏng (chìm xuống đáy, thay chỗ nước).

- Tìm chỗ trũng kiểu MC: `BlockDefinition.fluidSlopeFind` (nước 4, dung nham 2) ⇒ `FluidFlow.SlopeDistance` — chỉ chảy về hướng tới chỗ đổ xuống gần nhất, không có thì loang đều; nước đang chảy nằm trên cột đang rơi (miệng hố) không loang ngang, chỉ nguồn loang. Ngoài bản đồ / chưa nạp coi là ĐẶC (không thì nước chạy về mép thế giới). Play Mode: bệ đá trên cao có 1 lỗ ⇒ nước chỉ chảy về phía lỗ rồi đổ thành cột.
- Nạp lại: `FluidFlow.ScheduleEdited` — `VoxelWorld.Apply` lên lịch lại mọi ô chất lỏng trong sổ sửa của hình ⇒ dòng chảy dở chảy tiếp (thay cho lưu lịch như MC).

**LỚP CHẤT LỎNG RIÊNG kiểu Hytale (2026-10-14, user duyệt "okay luôn duyệt" — đầu bước 8):** chất lỏng KHÔNG còn nằm trong mảng khối.
- `VoxelChunk` có mảng chất lỏng song song theo lát (`GetFluid/SetFluid/NoFluidFast`, giá trị = số hiệu khối chất lỏng có nấc); `TopSection` tính cả lát chất lỏng; `Opacity/Emit` của ô = max(khối, chất lỏng) ⇒ `VoxelLightField.OnCellChanged(x,y,l,oldOp,oldEmit)` thay `OnBlockChanged`.
- `VoxelChunkSet`: nền sinh nước/dung nham vào lớp chất lỏng (khối ở đó = không khí); `GetFluid/BaseFluidAt/ApplyFluidEdit` + `_underFluid`. Ghi khối không `holdsFluid` ⇒ đẩy chất lỏng ra (nhớ chất lỏng gốc). BẪY đã dính: `ApplyFluidEdit(null)` khi ô đang có khối đặc KHÔNG được bỏ chất lỏng gốc đã nhớ (không thì đập đá giữa hồ ra ô khô vĩnh viễn).
- Sổ: `BlockStore` có sổ chất lỏng riêng (`SetFluid/RemoveFluid/TryGetFluid/AllFluids`, "" = cạn), cùng sự kiện `Changed`. `BlockPlacementOps.TryFluidAt/WriteFluid` (trùng chất lỏng gốc ⇒ bỏ mục); `Write` khối không ngâm được tự ghi chất lỏng "". `IBlockTerrain.TryBaseFluid`, `IBlockTickWorld.FluidAt` (mặc định không có).
- Save: `BlockIO.Format` 2 + `SaveData.fluids`; save format 1 tách bằng `BlockStore.SplitFluids` (mục chất lỏng ⇒ khối "" + chất lỏng; mục "" / khối không ngâm được ⇒ thêm chất lỏng "").
- `BlockDefinition.holdsFluid` (+ `BlockIds.HoldsFluid`, không khí = true): chưa khối data nào bật — dành cho bậc/rào/cửa bước 8. `FluidFlow` viết lại theo lớp riêng (`Enterable/CanFlowInto/Passable`), hoa cỏ không ngâm được vẫn bị cuốn.
- Test Fluid 41 (+ Layer_*, Holds_*, Lake_SolidInLakeLogsDry/BreakStoneRefills, Save_*); Physics 36, Placement 85, Light 41, Ticks 28, Roof 15, MultiFloor 45, BuildPersist 46, RenderSep 31. Play Mode: 9 ô world, 9087 ô chất lỏng, 0 trong lớp khối, loại ô y như trước (2062 sâu + 1965 nông); đá vào hồ ⇒ khô + tối, đập ⇒ hồ tràn lại thành nguồn, sổ sạch; xô múc/đổ, save format 2 khứ hồi 65 mục; lưới nước 46 chunk vẽ đúng.

**Nợ đã biết:** khối `holdsFluid` chưa chặn theo mặt (phiến/bậc chặn nửa ô) — làm cùng khối hình dạng bước 8; dòng chảy chưa đẩy người/vật phẩm.

Liên quan: [[project_world_like_minecraft_roadmap]], [[project_block_world]], [[project_block_light]], [[project_kinematic_body]], [[project_multifloor_pathfinding]].
