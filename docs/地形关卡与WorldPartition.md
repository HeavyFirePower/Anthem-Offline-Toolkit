# Anthem 地形、洞穴和 World Partition

这一步在 UE 5.8.4（Win64，CL 58902052）重建了 Dungeon_003 的两个局部场景。一张包含蜘蛛洞坡道，另一张包含洞底和洞壁。两张地图均已保存，并在独立编辑器进程中按源数据回读。工具直接使用已有的 `catalog.sqlite` 和 `raw`。

## 在游戏目录运行

```powershell
# 默认样本。导出高度、洞穴 FBX、图层资料，创建地图并回读
.\Run-Anthem.ps1 -Mode Terrain -Destination .\output\terrain\dungeon003 -ImportUE

# 洞底与洞壁样本。指定源场景及要恢复的两个刚性模型
$caves = @(
    'environments/caves/structures/meshes/cav_str_cavefloor20m01_mesh'
    'environments/caves/structures/meshes/cav_str_wall01_20m_mesh'
)
.\Run-Anthem.ps1 -Mode Terrain -TileGuid '2a666ba5-7f08-8157-0216-bda7f7c4c491' `
    -SceneName 'levels/dungeons/dungeon_003/static_content_zs_4' -CaveMesh $caves `
    -Destination .\output\terrain\dungeon003_cave -ImportUE
```

只做离线提取时，省略 `-ImportUE`。可通过 `-Output '实际解包目录'` 指定数据源，通过 `-EngineRoot` 指定 UE 安装目录。导出洞穴 FBX 时，工具会使用 UE 随附的 FBX 2020.2 运行库；自定义安装可将 `ANTHEM_FBX_SDK_BIN` 指向包含 `libfbxsdk.dll` 的目录。

在验证工程的内容浏览器中打开 `/Game/AnthemValidation/Terrain`。地图名中含 `e19c2849` 的是坡道，含 `2a666ba5` 的是洞底和洞壁。在 World Partition 编辑器中选中地形区域并加载。可在大纲中选中 `Anthem_Cave_65` 或 `Anthem_Cave_49`，按 F 定位洞底或洞壁。目标图层按源序号及完整对象 GUID 命名，当前未绘制源权重。

## 选择自己的瓦片

```powershell
.\Run-Anthem.ps1 -Mode TerrainList
.\Run-Anthem.ps1 -Mode TerrainAudit -Destination .\output\terrain
$audit = Get-Content .\output\terrain\terrain-audit.json -Raw | ConvertFrom-Json
$tree = $audit | Where-Object { $_.tree.name -like '*forttarsis*' -and $_.verified.Count -gt 0 } | Select-Object -First 1
$tile = $tree.verified | Select-Object -First 1
.\Run-Anthem.ps1 -Mode TerrainTiles -AssetId $tree.tree.id
.\Run-Anthem.ps1 -Mode Terrain -AssetId $tree.tree.id -TileGuid $tile.guid -TerrainNode $tile.node
```

`TerrainList` 显示目录记录 ID、名称、RID。`TerrainTiles` 显示已定位的外部高度块 GUID、节点号、世界 X/Z 起点和宽度（米）。`TerrainAudit` 按树列出 `verified`、`skipped` 和失败原因，检查高度与可用邻边。洞穴和图层状态须查看具体导出的清单。自选瓦片可另外传入同一关卡的 `-SceneName` 与一个或多个 `-CaveMesh`。工具只放置选定场景中原点位于导入区域内的对应实例。

## 本机全量目录覆盖

17 条 StreamingTree 涵盖 10 个地形名称。连同 Data/Patch 重复记录统计，共定位 1,414 个外部高度节点，1,395 个通过离线高度校验。下表按地形名称、世界起点、瓦片宽度、Chunk GUID 去重，同位置的不同数据版本分别计数。UE 导入验收单独统计在文末。

| 地形 | 定位 | 高度通过 | 其中全部分区极值精确复算 |
| --- | ---: | ---: | ---: |
| Nexus_001 | 208 | 208 | 169 |
| Dungeon_001 | 43 | 42 | 24 |
| Dungeon_003 | 70 | 70 | 49 |
| Dungeon_005 | 208 | 207 | 207 |
| Dungeon_005v3 | 16 | 16 | 16 |
| Dungeon_007 | 37 | 37 | 22 |
| Region_01 | 304 | 292 | 196 |
| Fort Tarsis | 14 | 14 | 7 |
| Tutorial_001 | 43 | 42 | 23 |
| Tutorial_002 | 138 | 138 | 103 |
| **合计** | **1,081** | **1,066** | **816** |

其余 250 个通过项使用保守高度包围范围，并有逐值匹配的非恒定同级邻边。15 个去重项仍有高度越界、邻边不一致或缺少可验证邻边的问题。表中只统计外部高度节点，未计入内嵌数据、碰撞或实例。

## 高度、坐标与边界

1. 从 StreamingTree 的三维包围盒和四叉树顺序恢复节点的世界范围，再按 Chunk GUID 和目录中声明的字节数解析数据。工具逐节点核对量化后的高度上下界，并校验原始文件 SHA-256。
2. 两个 Dungeon_003 样本的高度区各为小端 265 × 265 个无符号 16 位值。相邻块共享 9 行或列，裁去四周各 4 个边界采样后，`interior.r16` 保存完整的 257 × 257 个格点。样本中 341 个分区的最小值和最大值均可由格点精确复算。部分较粗 LOD 使用保守包围范围，工具会另外要求存在逐值相同、且具有起伏的同级邻边。高低 LOD 接缝另行处理。
3. UE 样本直接取中心 253 × 253 个格点，保留原间距，无插值。地图包含 4 × 4 个 63-quad Landscape Component，每 2 × 2 个组件分配到一个 LandscapeStreamingProxy。源瓦片覆盖 256 × 256 米，导入部分覆盖 252 × 252 米。`terrain_dungeon_05v3` 的 73 × 73 布局输出 65 × 65 原始内部格点，UE 输入取 64 × 64；本机该布局的实测瓦片高度恒定。
4. Frostbite 的 X/Z 映射到 UE 的水平 X/Y，源 Y 映射到 UE Z。样本世界高程为 `q × span_m / 65535` 米。UE Landscape 以 32768 为零点，工具同时设置 Actor 的 Z 偏移和 Z Scale。每块的起点、间距、垂直倍率和源范围均在 `manifest.json` 中。

## 洞穴、图层与 Shader 组合

洞穴提取沿 StaticModelGroup 的外部文件 GUID 和对象 GUID 找到 RigidMeshAsset，再通过 RID 找到 MeshSet 和 LOD0 顶点、索引数据。工具合成组与实例的变换，保留 FBX 原型，在 UE 中按原位置、旋转和缩放放置。验收检查重新载入的网格绑定、三角形数、本地包围盒和三个实例基向量。

图层提取从 TerrainData 的有序对象引用恢复 19 个 SingleTerrainLayerData，保留各自的 GUID 与散布类型引用。TerrainData 的三个 RID 分别连到 StreamingTree、VisualTerrain 和 TerrainLayerCombinations。`combinations.json` 将 Dungeon_003 的 **321 个组合**逐一关联到参与的图层序号和六个源数组槽中的 Shader 路径。该关卡共解析出 301 个不同的槽内 Shader 路径，另有一个默认 Shader。组合号和数组槽不能单独确定某个地面像素的材质。

导入器为 19 个身份各建一个 LandscapeLayerInfoObject，在 Landscape 上登记并回读。逐像素混合权重、洞口 Visibility Mask、程序化散布、位移和最终地表着色仍待解码。洞底、洞壁和坡道属于关卡摆放的网格，单值高度场只能表达一个水平坐标对应的一个高度。

全目录的 17 条树记录均能通过 TerrainData 的 RID 找到图层和组合表。每条记录有 1 至 22 个有序图层。Region_01 的两个版本分别有 6,151 和 6,317 个组合，均出现了超出 22 个 EBX 图层索引范围的编号 22，工具将它保留在 `unmapped_layer_indices`，等待确认其用途。Dungeon_003 的 321 个组合所用图层编号均能落在本关卡的 EBX 数组内。

## 两个 UE 样本的验收

| 检查项 | 蜘蛛洞坡道 | 洞底与洞壁 |
| --- | --- | --- |
| StreamingTree 节点 / 源场景 | 63 / static_content_zs_1 | 30 / static_content_zs_4 |
| 高度输入与回读 | 64,009 个 uint16，SHA-256 相同 | 64,009 个 uint16，SHA-256 相同 |
| 源邻边逐点验证 | 3 条，每条 2,385 个值 | 4 条，每条 2,385 个值 |
| 洞穴几何 | 坡道 6,176 三角，3 个实例 | 洞底 176 三角、洞壁 550 三角，各 1 个实例 |
| 图层回读 | 19 个 | 19 个 |
| World Partition | 4 个空间加载代理及其外部包 | 4 个空间加载代理及其外部包 |

每个样本目录中的 `ue-map.json` 给出内容浏览器路径。`ue-validation.json` 记录引擎版本、源数据与回读哈希，以及重新打开地图后取得的分区 Actor 描述符。`ue-readback.r16` 是从已保存的 UE Landscape 读取的实际高度。`layers.json` 和 `combinations.json` 记录图层与着色器来源；解析失败的可选资料会记录原因。`tiles.json` 给出所选树的瓦片坐标与原始 Chunk 文件。

完整关卡仍需按分辨率拼合父子瓦片，处理边缘、洞口、其他场景组和游戏原有的流送条件。目前的验收覆盖持久化、空间加载代理和独立进程回读，尚未测量移动玩家触发的运行时流送或 HLOD 切换。

本机安装版 UE 的原生导入插件及源码位于 `ue/AnthemValidation/Plugins/AnthemTerrainEditor`。重编译时使用目标引擎的 `RunUAT.bat BuildPlugin`，以该目录下的 `.uplugin` 作为 `-Plugin`，另一个空输出目录作为 `-Package`，传入 `-NoTargetPlatforms` 构建 Win64 编辑器模块，再替换插件的 `Binaries/Win64`。

UE 高度图导入的尺寸与比例约束见 [Epic Landscape 高度图文档](https://dev.epicgames.com/documentation/en-us/unreal-engine/importing-and-exporting-landscape-heightmaps-in-unreal-engine)。Frostbite 程序化地形混合的设计背景见 [DICE 的地形渲染论文](https://media.contentapi.ea.com/content/dam/eacom/frostbite/files/chapter5-andersson-terrain-rendering-in-frostbite.pdf)。
