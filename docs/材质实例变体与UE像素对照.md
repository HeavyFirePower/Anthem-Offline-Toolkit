# Spacebear 材质实例、着色器变体与 UE 像素对照

本页使用已解包的 Anthem PC Patch、共用 F9 Shader 数据库和 Unreal Engine 5.8.4。选取的是 Spacebear 默认身体材质的一个四渲染目标像素程序。可以从模型材质对象追到精确的 VS、PS 与参数表，也可以让游戏原始 PS 与 UE 对同一输入分别出图。

## 手动复现

在含 `Run-Anthem.ps1` 的目录运行。已有全量解包可直接使用。默认示例的结果位于 `<Destination>/45372f5a3c5d2819/`。

```powershell
# 先查 Spacebear 的基础材质、替换外观及每个外观能使用的具体程序
.\Run-Anthem.ps1 -Mode MaterialTrace -Destination output/material-study

# 从 states.jsonl 选择一个状态，单独导出它的 VS/PS 等已映射阶段及参数表
.\Run-Anthem.ps1 -Mode MaterialTrace -StateIndex 15745 -Destination output/material-study

# 原始 DXBC 在本机 D3D11 显卡上离屏运行。加 -ImportUE 建立和重开材质、渲染、出对照图
.\Run-Anthem.ps1 -Mode MaterialSample -Destination output/material-study -ImportUE

# 查另一个网格时，传该网格在目标 bundle 下的 EBX 资产 ID
.\Run-Anthem.ps1 -Mode MaterialTrace -AssetId '<mesh owner EBX id>' -Destination output/material-study
```

可以用 `-Output` 指向含 `catalog.sqlite` 和 `raw` 的目录，用 `-EngineRoot` 指定 UE 5.8.4 安装目录。默认本机硬件 D3D11 设备用于执行原始程序。UE 开两个离屏编辑器进程，第一个创建资产，第二个读回父材质、实例、纹理和像素。工具包附有回放器及 C++ 源码，运行样本无需另装编译器。

| 文件 | 可检查的结果 |
| --- | --- |
| `trace.json` | EBX 文件与对象 GUID、基础参数、每条 variation 的来源 bundle、字节偏移、替换对象、SurfaceShaderGuid 和 SurfaceShaderId |
| `states.jsonl` | 每行一条匹配状态，含几何声明哈希、原始 32 字节状态、solution 编号、各已映射阶段的 permutation key、程序偏移、长度和 SHA-256 |
| `state-15745/` | 指定状态导出的真实各阶段 DXBC 与反汇编、VS/PS 常量表、对应材质和外观记录。当状态不属于该 owner 时工具会报告错误 |
| `sample.json`、`selected.asm`、`selected.dxbc` | 本次选中状态、参数默认值／覆盖值／反射字节偏移、纹理来源、两个实验输入与实际执行的程序 |
| `surface-native/`、`surface/` | 表面材质用的全 mip 原生 DDS 与 UE 导入用的 BGRA8 解码 DDS |
| `ue-build.json`、`ue-render.json` | 编辑器版本、重开的资产名、表面纹理宽高与 mip 数 |
| `comparison.png`、`comparison.json` | 四列依次为 RT0 到 RT3。每种条件占三行，依次为原始 DXBC、UE 和 RGB 差值放大 100 倍。上半部为基准，下半部为命中实验。显示图做了色调压缩，数值报告比较未压缩的 RGBA float32 |

## 从模型实例找到编译变体

样本网格为 `enemies/creatures/sbr_spacebear/sbr_spacebear_model_mesh`。工具首先匹配同一 layout、package、superbundle 和 bundle 的 `meshvariationdb_win32`。随后以网格导出对象和材质导出对象的 GUID 对定位条目。按资产名字任取一份 Data 或 Patch 会混入其他外观。

| 默认身体的证据 | 此安装的值 |
| --- | --- |
| 材质对象 GUID | `5d1b0b91-e312-44ef-9a89-e01289c2f59a` |
| ShaderGraph | `shaders/characters/creature_sbr_sss_osm_core` |
| SurfaceShaderGuid / SurfaceShaderId | `423465db-266f-41ce-b1ef-d249e4c8162e` / `0x9c9fc922` |
| 选中状态 | `15745`，几何声明 `0xb893a4dc`，VS permutation `60928`，PS permutation `6109` |
| 像素程序 | 共用 F9 库偏移 `160910525`，14,072 字节，SHA-256 `080c088b265d33c0f3920b177464518c57aa2b3901030e7d3aab39302065b002` |
| 参数表 | VS 表 `15408`，PS 表 `6518` |

这个 bundle 的 6 个材质对象、12 个外观键产生 72 条材质映射，涉及 20 个 SurfaceShaderId。19 条带替换对象的映射都按 GUID 找到了所属外观的参数对象，来自 6 份外观 EBX。20 个 ID 在这份数据库中合计关联 30,812 条状态，按程序内容去重后为 1,134 个 VS、288 个 PS。默认身体单独占 3,780 条状态和 30 个不同 PS，这次选的 PS 被其中 120 条状态复用。选取状态 15745 是可复现实验设置，无法从安装文件知道某个历史画面选择了哪一条。

工具校验整份数据库 SHA-256、solution 与 state 的记录边界、permutation 的池索引、DXBC 长度和阶段。F9 样本将程序池、permutation、solution 和 state 分表保存。其他版本的布局仍需验证。不同数据库版本的程序不按裸哈希拼接。字段和旧版布局的交叉参考见 [Frosty 的 ShaderDbDump](https://github.com/CadeEvs/FrostyToolsuite/blob/1.0.6.3/FrostyCmd/ShaderDbDump.cs)。

## 实例值、默认值和运行时输入

身体基础实例保存了 7 项向量记录、5 个纹理槽和 2 项带资产引用的条件值。选中 PS 的反射有 22 项 external 常量，其中 12 项参与运算。按名字连接 EBX 和常量表，按反射的字节偏移和声明长度写入 `cb1`。例如 `EmissiveColor` 位于字节 32，实际写 3 个 float。字节 80 的 `EmissiveIntensity` 在数据库中默认是 1，身体实例把它改为 50。`DetailUVScale` 出现在实例中，此 PS 没有对应的反射字段。报告保留两侧各自的数据。

5 个实例纹理槽使用实际的 TextureAsset GUID 对和 RES RID。另有 4 个纹理按选中常量表的记录和路径哈希复核，分别为命中渐变、Bayer 图、云噪声及灰色反射率色块。`BloodTexture` 在所选实例和常量表里没有实际纹理映射，本次绑定黑色色块。改变 BloodDamage 在此测试中不会产生血迹。数据库还保留 s0/s1/s2 的滤镜、寻址和 LOD 设置。

基准条件取已保存的实例覆盖值和该程序的默认值。第二组把 `HitEffect` 改成 0.8，把弱点向量改为 (1, 0.5, 0.25)，把 `BloodDamage` 改为 0.4。光照用的 9 个实例向量、相机、时间及曝光在两侧显式赋值，可在 `sample.json` 和各 case 的二进制缓冲中复核。这两组值用于检验参数改变后的像素响应。原作历史帧中的常量缓冲、实例向量、曝光和动态纹理绑定仍未知。

## UE 资产及视觉验证

打开 `ue/AnthemValidation/AnthemValidation.uproject`，在内容浏览器进入 `/Game/AnthemValidation/MaterialStudy/PS_080c088b`。

| 资产 | 用法 |
| --- | --- |
| `MI_SpacebearBody` | 默认身体实例，挂到身体对应的材质槽。可编辑发光颜色、强度、命中、弱点、毛色和纹理 |
| `M_SpacebearBody` | 表面材质。参数节点、纹理对象与切线基节点连接到含可编辑 HLSL 的 Custom 节点。表面输出连接 Base Color、Normal、Roughness、Emissive，光照使用 UE Default Lit |
| `M_SpacebearBuffers` | 诊断材质。保留选中 PS 的全部四个输出，允许单独读取各目标的 RGB 与 alpha，不经过 UE 场景灯光和色调映射 |

表面材质使用实际解析出的全 mip 纹理，身体四张 4096² 主图可在编辑器直接查看。诊断材质与原始程序共同使用 128×128 网格和点采样。身体四张主图固定在源 mip 5，其余贴图使用源 mip 0。两侧用相同的像素中心、UV、基底、纹理传递函数、相机、参数和实例缓冲。差值按每个像素每个浮点分量计算。工具会在任一分量绝对误差超过 0.001 时保留图和报告并返回失败。

| 输入条件 | RT0 最大误差 | RT1 最大误差 | RT2 最大误差 | RT3 最大误差 |
| --- | ---: | ---: | ---: | ---: |
| 基准 | 0 | 0 | 0 | 0 |
| 命中与弱点覆盖 | 0 | 0 | 0 | 0.00000190735 |

一次真实差异来自渐变的纵坐标。程序在 `r3.y=0.5` 采样命中渐变，而虹彩渐变使用 `r3.w=0`。两者取同一行会让 RT1 偏色，即使普通大小的预览看起来接近。

四个目标的验证覆盖所选像素程序在上述条件下的行为。UE 表面采用该程序的部分输出适配 Default Lit。原作 VS 逐顶点运算、蒙皮与切线插值、Frostbite GBuffer 消费端、皮下散射、实际光照和后处理仍会影响最终外观。其他 29 个身体 PS 暂可追踪，尚未逐个建图和回放。UE 中的编辑与目标读回接口见 [Epic MaterialEditingLibrary](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/MaterialEditingLibrary) 和 [Epic RenderingLibrary](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/RenderingLibrary)。

数据库覆盖范围、其他材质和已回放的光照消费端见[材质数据库、光照与皮下散射](材质数据库光照与皮下散射.md)。
