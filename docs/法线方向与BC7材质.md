# Anthem 切线法线、图集与反射探针

## 浮雕法线如何接入 UE 5.8.4

本工具导出的 Freelancer 大厅地板 FBX，在浮雕平面实测源图为 **OpenGL 式 Y+**。以图像上缘为起点、V 向下计，增大的绿色沿图像上方倾斜。在当前 FBX 导入方式下，UE 材质需要把解码后的 Y 乘 −1。原作使用 D3D11，DDS 头只声明像素格式；这里只根据实际 VS、网格 UV 和 UE 世界法线判断方向。

| 核对项 | 实际结果 |
| --- | --- |
| Patch 浮雕 VS `53a46953…` | 从顶点属性计算 `T = -Tstored`、`B = cross(N,T) × BinormalSign`。浮雕 90 个有效三角形的源 B 与增大 V 的方向均反向，单位向量点积中位数 −0.991。砖地 144/144 个反向。黄铜 88/90 个反向，中位数 −0.990。 |
| UE 的浮雕平面 | 在已导入地板上依次输出切线空间三个单位向量，用 SceneCapture 世界法线回读。源 B 映至 UE 世界 +Y，UE 的正副切线实测约 (0.0143, −0.9995, −0.0010)。生产材质在这里翻转一次 Y。 |
| 原作贴图 | 浮雕、旧黄铜、砖面、底层石面、Spacebear 身体五张源图均为 BC5_UNORM。检查五张完整 mip 链，解码 BGRA8 后 B 恒为 0、A 恒为 255。PS 取 R×A 与 G，Z 取正根。 |
| 保存的 UE 资产 | 五张表面法线用线性 BC7。导入时关闭 sRGB 和 Flip Green Channel，保留现有 mip。Texture Sample 使用 Linear Color。Spacebear 已重开验证压缩和程序回放，角色蒙皮各部位的世界法线仍待逐处核查。 |

手动处理单张已确定用途的法线，在工具根目录或游戏目录运行：

```powershell
.\Run-Anthem.ps1 -Mode Texture -Match 'environments/praetorian/props/textures/prt_prp_mural05_norm' -NormalLayout ra-g -NormalY -1 -ImportUE
```

生成物在验证工程 `/Game/AnthemValidation/TextureStudy/BC7Normals/`。打开对应的 `M_…` 可编辑纹理参数 `NormalRG` 与标量 `TangentYSign`。这条命令已经在 2048×1024 的浮雕上跑通，重开读到平台格式 BC7，12 级导入源 mip 逐级哈希一致。平台压缩的角度误差见下表。

`-NormalLayout ra-g` 将原作 X 编码 R×A 预存入 R，G 原样写入，B 清零，A 写满。`rg` 用于程序直接取 RG 的正半球法线。UE 材质运行时的计算如下。

```hlsl
float2 xy = 2.0 * EncodedRG.rg - 1.0;
xy.y *= TangentYSign;
return float3(xy, sqrt(saturate(1.0 - dot(xy, xy))));
```

对接其他来源网格时，按该网格导入后的副切线指定 `-NormalY -1` 或 `1`。源槽的解码决定能否使用此公式。例如藤叶 `NOT_Mask` 的 B、A 带有反射与叶片参数，其候选 PS 以 Z=1 建立方向并归一化。烘焙光照、顶点动画和打包遮罩需保留其各自的分量。BC7 是目标平台设置，材质中的 B 是有符号切线向量的 Z，直接接 Normal 输入。

单独输出 BC7 DDS 时，先用 Texture 模式得到 `decoded/source.dds`，再指定另一条输出路径。工具保留输入 mip 数。

```powershell
python tools/normal_rg.py .\decoded\source.dds .\normal-bc7.dds --layout ra-g --bc7
```

## 压缩后在网格上的误差

从 UE 保存的实际 BC7 表面纹理各读取一个最长边为 128 的源 mip，逐纹素计算与原始解码的单位法线夹角。无损的四目标原作 PS 对照仍单独使用原始采样输入。

| 表面 | mip | 纹素 | 平均夹角 | P95 | 最大夹角 |
| --- | ---: | ---: | ---: | ---: | ---: |
| 浮雕 | 4 | 8,192 | 0.37° | 1.01° | 3.23° |
| 旧黄铜 | 4 | 16,384 | 0.03° | 0.45° | 1.01° |
| 砖面 | 5 | 16,384 | 0.82° | 2.29° | 11.28° |
| 底层石面 | 5 | 16,384 | 0.94° | 1.86° | 4.60° |

这四次 GPU 回读使用同一 UE 动态材质实例，通过 `set_texture_parameter_value` 依次给 `NormalMap` 绑定四张实际 BC7 纹理，分别与各自源图对照。原作何时换绑、由哪个事件驱动，以及当帧的常量缓冲仍待验证。

在保存的三槽地板关卡重拍两台相机。录像浮雕留出区近景 26,870 像素的显示 RGB MAE 为 0.07368，远景 3,885 像素为 0.09049。上一版为 0.0754 和 0.0939。两轮均按同一流程拟合场景灯光，差值包括法线方向、压缩和重新拟合灯光的共同影响。115524 的设备与 115452 的砖墙沿用照明参考区，两帧尚未定位到唯一网格与材质。复现、相机和输出说明见[录像画面与地板材质](录像画面与地板材质.md)。

## 查询每个程序的通道用法

```powershell
.\Run-Anthem.ps1 -Mode TextureVariants
.\Run-Anthem.ps1 -Mode TextureLookup -Match 'environments/praetorian/props/textures/prt_prp_mural05_norm'
.\Run-Anthem.ps1 -Mode TextureLookup -All
```

在同一输出目录已有 TexturePixels、TextureConsumers 索引时可直接查询。七组所有者和数据库合计 45,752 条候选状态，575 个不同 PS。23,673 个 Texture 名称中，30 个至少有一种用法追到具体程序并解释了 RGBA，比上一轮多 12 个。23,643 个名称仍缺逐程序通道语义。查询结果保留实例、RID、寄存器、指令、表面键和候选状态。下表给出本轮接入反查的两种程序。

| PS 与槽位 | 对应纹理及分量 |
| --- | --- |
| `cd984782…` 的 Diffuse／Normal／AMMS | 浮雕和旧黄铜各三张。Diffuse RGB 乘实例色调。Normal 取 R×A、G 并重建正 Z。AMMS 的 R 缩放间接光，R×G 进入 `0.5*saturate(1.4*R*G)`，B 为金属遮罩，A 为平滑度。 |
| `a57f7881…` 的 BaseColor／BaseLayer_BaseColor | 砖层与粗石层各有独立颜色、法线和 AMMS。两张颜色图取 RGB。底层石面颜色图的 A 另与顶点红色、MaskStrength 一起形成图层和砂浆边权重。 |
| 同一砖地 PS 的两张 Normal 和两张 AMMS | 每层以 R×A、G 重建法线，砂浆分支修正底层 XY，混合后归一化。两张 AMMS 按层权重混合，R 与 R×G 参与间接光和反射项，B 控制金属量并选择平滑度参数区间，A 在区间内插值。 |

## 材质库默认纹理、图集与未解析的导出

```powershell
.\Run-Anthem.ps1 -Mode TextureEvidence -Match 'vfx/textures/atlas/debris/vfx_debris_blurred_fb_2x2'
.\Run-Anthem.ps1 -Mode TextureLookup -Match 'shaders/swatches/swatch_grey'
```

`TextureEvidence` 默认写入已解包目录下的 `converted/texture-coverage`。省略 `-Match` 只生成审计，图集样本也可用报告的 `res_id` 指定。工作区本轮实测报告放在 `output/texture-coverage`。

- **默认纹理哈希。** `hash-defaults.json` 遍历九库的静态常量块，记录 199,569 个纹理项和 595 个不同的 32 位路径哈希。595 个均在本安装的 Texture／AtlasTexture 逻辑名称中唯一命中。每个库和寄存器保留使用次数、常量块和字节偏移。例如 `b06fb57f` 指向 `shaders/swatches/swatch_grey`，出现 2,980 次。相同名称的 Data／Patch 可能有不同内容。实际帧是否覆盖默认槽，仍须运行时证据。
- **图集。** `atlas-grids.json` 将 77 个独立 AtlasTexture 名称的 EBX 对象、RES RID、帧数和列数追到 77 个唯一 chunk。全部可列出规则 UV 网格，73 个可精确分为整数像素矩形，其中 52 个为多帧。另 4 个的行高或列宽落在分数像素处。报告保留 `uv_cell_size`、边框、标志位以及独立的 `rectangles`。图集包含 VFX 等资产，统计单位与 UI 行的 808 条目录记录不同。样本的 256×256 顶级 mip 按 BC3 解码，导出四张 128×128 DDS，从磁盘重读后拼回，65,536 像素全部一致。编号按从左到右、从上到下；具体效果的播放时序、内侧边框与动态寻址仍待核实。[Frosty 图集头与边框](https://github.com/CadeEvs/FrostyToolsuite/blob/1.0.6.3/Plugins/AtlasTexturePlugin/AtlasTexture.cs) · [Frosty 图集网格与 BC3 显示实现](https://github.com/CadeEvs/FrostyToolsuite/blob/1.0.6.3/Plugins/AtlasTexturePlugin/FrostyAtlasTextureEditor.cs)
- **未解析的导出。** 若已有 `texture-graph.sqlite`，工具生成 `unresolved-exports.json`。190 对尚未解析的导入 GUID 中，179 对在正确文件 GUID 的实例数据区找到了对象 GUID 字节，11 对未在这些失败样本中定位。报告给出文件和偏移。后续须验证对象边界、类型和索引，这些候选不会计入已解析导出。

## MES_005 探针投影与混合

```powershell
.\Run-Anthem.ps1 -Mode TextureVisual -ImportUE
```

昼夜两层共 82 个已放置盒，其中 18 个写有 `UseProxyReprojection=true`，64 个为 false，82 个的 Enabled 都为 true。报告还记录六个方向的淡出值、捕获淡出距离、影响区扩张和淡出距离。日间对象 32 与夜间对象 34 的变换和局部偏移相同，各绑定一张独立的 128×128 RGBA16F 立方体，仅有 mip 0。

`M_blend` 用盒局部位置和反射方向求射线出盒交点，减去烘焙捕获偏移，分别采样昼夜源图后线性混合。实验取缩放后单位盒的半长作为边界，测试 3 组内部位置和方向、投影开／关、混合权重 0、0.25、1。18 组 UE 浮点目标重开回读与独立 CPU 源纹素计算的最大 RGBA 差为 0。另一个有多级 mip 的盒仍通过 120 个方向采样检查。`visual/visual-render.json` 保留每组入参、期望值和读数。代理投影标志来自原始 EBX，盒边界解释、采样滤波和昼夜权重是本次重建条件。原作的投影程序、探针优先级、邻盒混合、逐帧权重以及大厅录制时的实际绑定仍缺直接证据。
