# Anthem 纹理格式、命名后缀与 RGBA 通道

浮雕、黄铜、砖地与 Spacebear 的 UE 表面法线已采用线性 BC7。单图导入命令、Y 符号和 RG 重建 Z 的适用条件见[法线方向与 BC7 材质](法线方向与BC7材质.md)。


## 从一张图追到材质

全量像素校验、模型以外的引用和按图反查命令，以及雨幕、关卡反射盒与加载图的 UE 实测见[纹理逐资产验收与场景采样](纹理逐资产验收与场景采样.md)。

在含 `Run-Anthem.ps1` 的游戏目录打开 PowerShell。已有完整解包时，可以直接查看 Spacebear 的皮肤材质。

```powershell
$dst = Join-Path $PWD 'output/texture-study'
$mesh = 'enemies/creatures/sbr_spacebear/sbr_spacebear_model_mesh'
$skin = 'enemies/creatures/sbr_spacebear/textures/sbr_spacebear_jungle_a'
.\Run-Anthem.ps1 -Mode TextureMaterial -Match $mesh -Destination $dst
.\Run-Anthem.ps1 -Mode Texture -Match "${skin}_osm" -Destination $dst
.\Run-Anthem.ps1 -Mode Texture -Match "${skin}_efc_mask" -Destination $dst
.\Run-Anthem.ps1 -Mode TextureChannels -Destination $dst
```

`TextureMaterial` 第一次会建立 TextureAsset 引用索引。`bindings_*.jsonl` 每行保留所属 EBX、材质对象、Shader 引用、参数槽和纹理引用。可以在 PowerShell 直接列出已解析的槽位。

```powershell
Get-Content "$dst/bindings_sbr_spacebear_model_mesh.jsonl" | ForEach-Object {
    $b = $_ | ConvertFrom-Json
    foreach ($t in $b.textures) { [pscustomobject]@{ Slot=$b.slot; Texture=$t.name; RID=$t.rid } }
} | Sort-Object Slot,Texture -Unique | Format-Table -AutoSize
```

样本中，身体的 `Diffuse`、`Normal`、`OSM_Mask`、`EFC_Mask` 分别指向 `_diff`、`_norm`、`_osm`、`_efc_mask`。另一个材质的 `Normal` 指向毛发纹理，`Diffuse` 指针为空。空指针、继承默认值以及动态替换都需要保留其上下文。引用链通过参数的文件／对象 GUID 找到 TextureAsset，再按其 RES RID 和逻辑名匹配 Texture 资源。不同内容版本保留在报告中。

`Texture` 接受完整逻辑名，也接受目录中的 `-AssetId`。在它打印的 `texture.json` 同目录打开 `channels.html`，可并排比较 R、G、B、A，单独的灰度图为 `R.png` 至 `A.png`。预览采用第 0 级 mip 的第一面、数组第一层或体纹理第一个 Z 平面，最大 512 像素，数值范围取自该平面的全部像素。HDR、深度、其他面和其他 mip 的原值应从 DDS 读取。`source.dds` 保留原始格式与像素块，`decoded/source.dds` 保存适合 UE 导入的完整解码副本。

## 文件名可以说明什么

目录类型 `.Texture` 表示 Frostbite 的纹理资源。文件名末尾的词来自内容作者或烘焙系统。下表数字按 **23,673 个不同逻辑路径**计数。工具把最后一个下划线后的词作为后缀，生成的 GUID 名称取末尾连字符后的词。例如 `_efc_mask` 计入 `_mask`，`_norm_a` 计入 `_a`。`suffixes.json` 保留全部 1,844 个分组、名称数、路径样例和代表性头格式；同一纹理头可服务多个路径。

| 后缀及路径数 | 从本次文件能确定的用途 |
| --- | --- |
| `_diff` 2,579、`_diffa` 40 | 常见基础色，已绑定到 `Diffuse` 和 `BaseColor`。部分 `_diff` 进入 `EmissiveRGB`，部分 `_diffa` 进入 `NOT_Mask`。透明裁切由具体 pass 决定。 |
| `_norm` 2,684、`_normal` 51、`_n` 89、`_nm` 23 | 常见法线或方向数据。`_norm` 多为 BC5，也有 BC3、BC7 和浮点图。`vfx/vertanim` 下的浮点纹理还用于顶点动画数据；要结合目录与绑定槽确定空间和解码方式。 |
| `_amms` 1,434、`_osm` 377、`_aomo` 186 | 实际槽包括 `AMMS`、`OSM_Mask`、`AO_MO`。三个程序的 B、A 用法见下表。`_osm` 也有进入 `AMMS` 的实例。 |
| `_cs` 176、`_nam` 67、`_nrh` 46 | 环境材质中可见 `CS`、`NAM` 以及 `Layer1/2/3CS`、`Layer1/2/3NRH`。支持查到实际层和程序中的采样分量。 |
| `_mask` 2,173、`_efc` 8 | `_mask` 同时覆盖 `EFC_Mask`、`AMMS`、`OSM_Mask`、`LeafClusterMask`、`SubSurfaceTexture` 等槽。Spacebear 的复合后缀 `_efc_mask` 已定位到 `EFC_Mask`。 |
| `_osa` 20、`_not` 14 | 植被材质中查到 `OSA` 与 `NOT_Mask`。分别有裁切和树叶变色／次表面相关的实测程序。 |
| `_emis` 53、`_emiss` 28、`_alpha` 7 | 自发光或遮罩的命名线索。`_emis` 可绑定 `Emissive_Map`；该槽在一个程序里通过参数选择 RGB 分量。名为 `Alpha` 的槽也可以读取纹理的 R。 |
| `_hght` 22、`_disp` 12、`_flowmap` 15 | 高度／位移／流向候选。高度样例有 R8 和 BC4，水体流图有 BC5。R8、BC4 只有独立的 R 数据，BC5 有独立的 R、G。高度单位、零点以及流向的轴和符号仍需对应使用者。 |
| `_icon` 874、`_iconhd` 262、`_hdicon` 110 | UI 图标或分辨率变体。形状、遮罩和透明度要按 UI 材质／混合状态确定。 |
| `_sto` 45、`_rngintcol` 42、`_01` 120、`_a` 29 | 存在装备贴花版本、编号或资产变体。例 `decal_t1_fierceheart_sto` 与角色眉毛 `_norm_a`；这些词本身没有定义 RGBA。 |
| `-bakedrvtex` 1,993 | 反射体积目录中出现的烘焙纹理，实际头为六面 RGBA16F Cube。 |
| `-iblgbufferbasecolorsss`、`-iblgbufferdepth`、`-iblgbuffernormalskyviz` 各 1,751 | 反射体积的基础色／次表面、深度、法线／天空可见性缓冲。类型与尺寸取自头；精确的打包、深度空间和采样面方向仍需追消费程序。 |
| `_radiositynormaltexture` 685 | 位于关卡烘焙照明路径的纹理。直接套用角色切线空间法线的还原公式会丢失其照明语义。 |

## Shader 实际如何使用 RGBA

下表逐项对应 `TextureChannels` 生成的同名汇编文件；工具检查源字节码 SHA-256、反射名和指令，并在 `evidence.json` 记录行号。数值是 **Shader 采样后的值**。sRGB 采样时，RGB 按视图转换到线性值，A 保持线性。源文件 RGBA、DXGI 解码补出的分量以及程序真正读取的分量应分别记录。

| 材质槽／证据文件 | R | G | B | A |
| --- | --- | --- | --- | --- |
| `Diffuse` · `diffuse_normal_amms.asm` | 基础色 R | 基础色 G | 基础色 B | 此 pass 用 A 裁切，小于 0.5 丢弃像素 |
| `Normal` · 同上 | 与 A 相乘后映射到有符号 X | 映射到有符号 Y | 未读，Z 从 X、Y 重建 | 参与 X；对没有独立 A 的 BC5，采样 A 为 1 |
| `AMMS` · 同上 | 调制间接光并与 G 相乘 | 反射率输出为 `0.5 × saturate(1.4 × G × R)` | 金属遮罩对应输出为 `1−B`，还参与另一打包值 | 原值进入平滑度对应输出 |
| `OSM_Mask` · `osm.asm` | 重映射至反射率对应输出，同时进入照明相关输出 | 在材质设定的端点间插值，叠加磨损／血迹项后将平滑度夹在 0 至 0.95 | 原值进入金属遮罩对应输出，也改变 G 的重映射区间 | 此变体未读 |
| `EFC_Mask` · `efc.asm` | 发光遮罩，乘 `EmissiveColor`，后续受强度控制 | 此变体未读 | 此变体未读 | 此变体未读 |
| `CS` · `cs_nam.asm` | 基础色 R | 基础色 G | 基础色 B | 在平滑度端点间插值，后续受湿润项影响 |
| `NAM` · 同上 | 有符号法线 X | 有符号法线 Y；此程序将切线法线 Z 设为 1 | 调制烘焙光照 | 进入随湿润项变化的反射率对应输出 |
| `AO_MO` · `exo_ao_mo.asm` | 调制照明 | 混合后的分量进入反射率对应输出 | 与外骨骼贴花金属参数混合 | 与贴花平滑度参数混合 |
| `OSA` · `osa.asm` | 未读 | 未读 | 输出 `saturate(2 × B)`，小于 0.5 裁切，即 B 小于 0.25 时丢弃 | 未读 |
| `NOT_Mask` · `not_foliage.asm` | 此变体未读 | 此变体未读 | 重映射后进入反射率对应输出 | 参与植被颜色变化和 `SSSR_SSST` 参数的运算 |
| `NOT_Mask` · `not_two_sided.asm` | 有符号法线 X | 有符号法线 Y；设 Z 为 1，法线根据正反面调整 | 同时调制间接光和反射率对应输出 | 与 `SSSR_SSST` 的两个分量分别运算 |
| `Layer1/2/3NRH` · `nrh_layers.asm` | 三层的法线分量参与混合 | 三层的另一法线分量参与混合 | 此变体未读 | 第 2、3 层用 A 作为图层混合除数，并设置正下限；第 1 层未读 A |
| `Emissive_Map` · `emissive_map.asm` | 与选择参数的 R 做乘积 | 与选择参数的 G 做乘积 | 与选择参数的 B 做乘积，三者求和后乘发光颜色 | 此变体未读 |
| `EmissiveRGB` · `emissive_rgb.asm` | 发光色 R | 发光色 G | 发光色 B；RGB 的和还用于强度计算 | 此变体未读 |
| `Alpha` · `alpha.asm` | 裁切值，小于 0.5 丢弃；同一程序另有独立的溶解测试 | 未读 | 未读 | 未读 |

`output_anchor.asm` 中，三个命名常量 `MetalMask`、`Reflectance`、`Smoothness` 分别写入 `o2.y`、`o2.z`、`o0.z`。表中的“对应输出”依据此程序与所列变体的输出位置对照推断。指令级运算在列出的程序内可复现；同名槽可以跨数个 pass，精确的材质实例到 DXBC 排列键仍待验证。套用到 UE 时，应从已确定的**最终平滑度**得到候选 Roughness，并保留原参数的上下限。金属工作流、反射率和 GBuffer 编码还需要对应 Frostbite 的着色模型，不能直接照抄输出缓冲字节给 UE 材质引脚。

Spacebear `_osm` 源头为 **BC1_SRGB**，首级 R、G、B 像素各覆盖 0–255，A 全为 255。`_efc_mask` 为 **BC1_UNORM**，R 为 0–239、G 为 0–16、B 为 0–255、A 全为 255。后一张图的 B 有完整动态范围，但所列 EFC 变体仅采样 R。把另一个通道标为毛发、涂层或其他物理量，仍需要读取它的程序佐证。

一个完全独立的例子是三维调色参考表 `systems/postprocess/colorcuberef_linear`。遍历 32³ 个体素确认 R 仅随 X、G 仅随 Y、B 仅随 Z 单调变化，端点为 0、255，各体素 A 恒为 255。每轴的实际 32 个取值记录在 `evidence.json`，可以据此重建采样格。

要追查表外的槽，可按反射名称搜索候选像素程序。例如在 PowerShell 执行：

```powershell
.\Run-Anthem.ps1 -Mode TextureChannels -Match NOT_Mask -Limit 2 -Destination $dst
```

默认查本安装的共用 Shader 库，按文件顺序取最多指定数量的不同程序。换库可加 `-AssetId` 指定已解包 IShaderDatabase 的完整目录 ID。`programs.json` 列出每个候选的偏移、SHA-256、纹理寄存器及实际访问指令；相邻的汇编可用于继续追踪采样值到输出。匹配区分大小写，空结果仅表示该库没有搜到对应反射名。要归属到具体材质，仍需对应的 pass 和变体键。

## 模型材质槽的普查结果

6,713 份按内容区分的所有者 EBX 共读到 **37,208 个材质纹理槽**。其中 32,946 个唯一解析到逻辑名和 RID，4,108 个可确定逻辑名但存在多个 RID，1 个涉及多个不同名字，23 个外部引用缺少匹配，130 个为空。可唯一确定路径的绑定覆盖 8,542 个纹理名。

仅统计名字和 RID 都唯一的槽位，`_osm` 绑定到 `OSM_Mask` 665 次、`OSM_mask` 100 次、`Specular` 2 次、`AMMS` 和 `Emissive` 各 1 次。`_diffa` 绑定到 `BaseColor` 299 次、`Diffuse` 54 次、`NOT_Mask` 1 次。这些是绑定出现次数；同一纹理可被多个模型或材质版本使用。

## 已观察的格式与维度

246,332 条已解包 Texture 记录中，按 RES 内容 SHA-256 去重得到 **24,414 份纹理头**。逐头校验了字段、各 mip 尺寸与目录中的 chunk GUID、完整长度和候选内容哈希数。全部找到唯一完整像素候选。下表的“UE 样本”按每个实际出现的维度／格式编号组合选一张。

| 源维度 | 记录数 | 不同纹理头 | UE 样本与保存的资产类型 | 本维度实际格式编号 |
| --- | ---: | ---: | --- | --- |
| 2D | 230,038 | 17,074 | 16 / Texture2D | 6、18、20、51、54、55、56、57、60、61、62、63、64、66、67、123 |
| 2DArray | 12,828 | 5,275 | 6 / Texture2DArray | 18、55、61、63、66、123 |
| Cube | 3,325 | 2,050 | 1 / TextureCube | 40 |
| 3D | 141 | 15 | 5 / VolumeTexture | 6、18、27、55、62 |

18 个 Frostbite 格式编号分别为 R8_UNORM（6）、RGBA8_UNORM／sRGB（18、20）、RGB10A2_UNORM（27）、RGBA16F（40）、RGBA32F（51）、BC1 线性／sRGB（54、55）、BC1A 线性／sRGB（56、57）、BC3 线性／sRGB（60、61）、BC4（62）、BC5（63）、BC6H 无符号浮点（64）、BC7 线性／sRGB（66、67）和 D16_UNORM（123）。BC1 与 BC1A 可使用相同的 DXGI 编码；工具另存原始编号及 sRGB 标志。当前目录没有 1D、1DArray、CubeArray、BC2 或有符号 BC6H 的验收样本。

数组和 Cube 的源像素按 mip 排列同一级的所有面／层。导出器转换为 DDS 所需的每面／层拥有完整 mip 链；3D 的 Z 深度随 mip 缩小。每个样本验证源 chunk SHA-256，并按面和 mip 检查 DDS 中的目标字节。DDS 扩展头记录 DXGI 格式、资源维度、Cube 标志和数组长度。[Frosty 纹理导出实现](https://github.com/CadeEvs/FrostyToolsuite/blob/1.0.6.3/Plugins/TexturePlugin/TextureExporter.cs) · [微软 DDS DX10 扩展头](https://learn.microsoft.com/en-us/windows/win32/direct3ddds/dds-header-dxt10)

为了在 UE 5.8.4 保存完整源像素，导入使用 DirectXTex 解码的 DDS。普通彩色纹理使用 BGRA8，BC6H／RGBA16F 使用 RGBA16F，RGB10A2／RGBA32F 使用 RGBA32F，D16 使用保留 16 位深度值的 G16。再次启动编辑器从保存的 UTexture.Source 回读，28/28 样本类型、sRGB 属性及全部 146 级 mip、各层／面的解码像素哈希一致。运行时派生压缩、立方体的原作世界朝向、原始 GPU 采样器和最终画面仍需分别验证。[DirectXTex DDS I/O](https://github.com/microsoft/DirectXTex/wiki/DDS-I-O-Functions)

## 手动复现和核对范围

```powershell
$dst = Join-Path $PWD 'output/texture-study'
.\Run-Anthem.ps1 -Mode TextureAudit -Destination $dst
.\Run-Anthem.ps1 -Mode TextureSamples -Destination $dst -ImportUE
.\Run-Anthem.ps1 -Mode TextureMaterial -All -Destination $dst
```

`TextureAudit` 写入 `texture-audit.json` 和 `suffixes.json`。`TextureSamples` 自行补建缺少的普查报告；重复验证不同安装时请指定各自的输出目录。`-ImportUE` 保存到验证项目的 `/Game/AnthemValidation/TextureStudy`。用 `ue-texture-readback.json` 的 `all_source_mips`、`mips[].equal` 和 `passed` 核对每份资产。若仅研究原始格式，可以省略 `-ImportUE`。

`TextureMaterial -All` 对全目录中与 MeshSet 同名的 EBX 做指定结构的普查；也可对单个材质预设用 `-Match 完整路径` 查询。`mesh-material-bindings.jsonl` 可按完整纹理名反查模型或参数，摘要 `mesh-material-bindings.summary.json` 按后缀和槽位给出实际出现次数。统计单位为按内容区分的所有者 EBX 中的槽位，包含多个材质及版本。UI、特效、后处理和运行时替换还各有其他纹理入口。全库每一个材质的 RGBA 用法尚未完成确认。
