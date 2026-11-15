# 焊接母效果与下落火花的 Niagara 转换

## 播放六个焊接实例

在已部署工具的游戏目录运行以下命令。省略 `-ImportUE` 只提取这份母效果的清单和纹理。已有全量解包时可以直接执行。

```powershell
.\Run-Anthem.ps1 -Mode FXComposite -ImportUE
```

在验证工程中打开 `/Game/AnthemValidation/FX/Welding/NS_Amb_WeldingTorch`。五份 Ultra 发射器模板生成六个 Niagara 发射器。源对象编号保留在名称中，便于在编辑器里单独预览。母效果的六个局部变换均无旋转，E4 的高度偏移为 −16.63 cm。

| Niagara 发射器 | 已接入的数据 | 可以检查的行为 |
| --- | --- | --- |
| E3、E5 主火花 | 同一份源 EBX，两个独立实例，各 200 粒/秒，上限各 50 | 从半径 5 cm 的球形区域向外发射，出生时选取 2×2 图集中的一帧 |
| E7 快速火花 | 独立速度范围，30 粒/秒，上限 21 | 同样使用四帧图集，可以在回读中检查比主火花更高的横向速度 |
| E4 下落火花 | 局部高度、20 粒/秒、单帧碎屑图集、向下初速与随机重力 | 起点低于主火花，沿 UE 负 Z 下落 |
| E6 耀斑 | 20 粒/秒、颜色和透明度曲线、原始 Diff_01 和 Mask | 加法精灵叠加在发射点，遮罩取 R 是本次重建的选择 |
| E8 烟雾 | 5 粒/秒、正 Z 加速度、五控制点透明度、原始 Diff_01 和 Mask | 在早期迅速变浓，随后衰减。可在 Particle Update 的 OverLife 查看曲线 |

**曲线与随机量。** 烟雾透明度使用归一化年龄，五个横坐标约为 0、0.13947、0.48529、0.73529、1，纵坐标约为 0、1.0297、0.6040、0.1782、0。实现将末端横坐标设为 1，将 G 数组的相邻值解释为每段 Hermite 的起止切线。每 1/256 寿命采样并额外保留节点。回读按独立的分段公式校验粒子 Alpha。这个比较能检验 Niagara 曲线的写入和插值，原作的切线定义仍待确认。

每个粒子出生时各抽取一次寿命和竖直加速度，按源比例在名义值两侧均匀采样。烟雾的理论范围为 1.538–2.462 秒及 +13.88 至 +26.12 cm/s²。下落火花为 0.3005–0.6995 秒及 −1242 至 −758 cm/s²。相同源随机数序列和概率分布尚无原作粒子记录可核对。模板的 MaxCount 用于限制每步可生成数。

**材质。** 烟雾追到 `fxs_p_diff_lit_rgb_uvdistdm_t` 的四个纹理槽，耀斑追到 `fxs_p_diff_rgb_uvdistdradial_cp_a` 的三个槽。清单保存 Shader 文件与导出对象 GUID、纹理来源、参数和通道选择。预览材质将主纹理与粒子颜色相乘，用遮罩调制 Alpha；烟雾使用半透明，其他层使用加法。两处主火花与快速火花共用 muzzle-flash 图集，下落火花使用 debris 图集。烟雾的场景照明、法线和 UV 扭曲、耀斑径向扭曲与曝光、二级透明度、风场、空气阻力及母效果灯光均待还原。

**验收文件。** `converted/fx/welding/welding.json` 记录六个对象的来源和变换。UE 保存系统后重新启动，按 1/60 秒步长推进 120 步，每 10 步读取一次 SimCache。`welding-playback.json` 的 `columns` 给出每粒数据的字段顺序，包含归一化年龄、寿命、线性 RGBA、竖直加速度、三轴速度、位置和图集帧。每帧按发射器分组，跨帧存活的粒子会重复计入观测数。`ue-reopen.json` 给出各实例的范围、误差和本次运行标识。`preview.png` 是固定曝光、关闭 Bloom 的独立 PIE 合成。曝光和机位用于检查叠层，尚无原作焊接特写可对齐。

本轮在 UE 5.8.4 的 12 个采样帧中累计回读 1,565 条粒子观测。E3、E5 的最大存活数均为 50，E3、E5、E7 各自读到 0–3 四个图集帧。烟雾的 58 条观测中，寿命为 1.618–2.220 秒，竖直加速度为 +14.24 至 +24.67 cm/s²，Alpha 对候选 Hermite 曲线的最大绝对误差为 0.00258；六路最大误差为 0.00556 以内。

另可打开 `sparks.png`、`flare.png` 和 `smoke.png` 查看独立回放。每组都用 SimCache 检查实际活跃的发射器。火花与合成曝光均为 −3 EV，耀斑与烟雾的检查图提高到 +4 EV 以显示暗部。各组与同曝光空白帧比较，RGB 至少一个通道差值超过 6/255 的像素分别为 2,801、38、3,129。独立回放重新抽取随机数，三张图不能相加来重算合成图。

## 单个下落火花的手动复现

在游戏目录或工具根目录打开 PowerShell。工具读取已解包的 `catalog.sqlite` 和 `raw`。指定其他解包位置用 `-Output`，指定结果目录用 `-Destination`。

```powershell
# 普查安装中 vfx/ 下的 EBX，列出发射器名称、版本及资产 ID
.\Run-Anthem.ps1 -Mode FXAudit

# 查看母效果里各发射器的局部对象编号和外部 GUID 引用
.\Run-Anthem.ps1 -Mode FXTrace -Match 'vfx/environmental/ambient/effects/vfx_amb_welding_torch_sparks'

# 查看样本的四档质量模板、处理器链及其曲线
.\Run-Anthem.ps1 -Mode FXTrace -Match 'vfx/environmental/ambient/emitters/emt_amb_welding_sparks_falling'

# 生成样本清单和曲线，再用 UE 5.8.4 建图、重开、播放并抓取粒子数据
.\Run-Anthem.ps1 -Mode FXSample -ImportUE
```

仅需离线分析可运行 `-Mode FXSample`。默认输出为解包目录的 `converted/fx`。多版本同名 EBX 用 `FXTrace -Match` 分别输出，需指定具体版本时使用 `-AssetId`。可用 `-EngineRoot` 指定 UE 安装目录。本工具调用引擎自带的 Niagara 和 Cascade to Niagara Converter 编辑器插件来编写原生 Niagara 模块，输入直接来自 Anthem EBX。运行已保存的系统只需 Niagara。使用固定的样本资产路径；重复执行会重开并检查已有系统。在 UE 中自行编辑该样本前可先复制一份。

| 输出 | 查看内容 |
| --- | --- |
| `audit/audit.json` | 按根类型统计的名称与内容版本，发射器条目含完整资产 ID 和 SHA-256 |
| `trace/<资产ID>.json` | 对象、偏移、GUID、四档质量模板、NextProcessor 顺序、Pre 求值器指针和输入字段 |
| `sample/emitter.json` | 母效果绑定、源参数、两条曲线的系数、颜色端点及 129 组采样键 |
| `sample/atlas.res`、`sample/atlas.chunk` | 根据发射器 BaseTexture 的 GUID 对、AtlasTextureAsset 的 RID、头内 Chunk GUID 和字节数导出的原始图集 |
| `sample/playback.json` | 每个采样帧中各存活粒子的归一化年龄、RGBA、寿命、尺寸、竖直速度和位置 |
| `sample/ue-build.json`、`sample/ue-reopen.json`、`sample/preview.png` | 两个编辑器进程的同次运行标识、误差、资产路径与 PIE 实际渲染 |

打开 `ue/AnthemValidation/AnthemValidation.uproject`，在内容浏览器进入 `/Game/AnthemValidation/FX`，双击 `NS_Amb_WeldingSparks_Falling` 播放。展开 `FallingSparks` 的 Particle Update，查看 `OverLife` 模块里的 RGB、Alpha 曲线和归一化年龄输入。将系统拖入自己的关卡可单独播放这个发射器。Niagara 的运行阶段与缓存数据说明见 [Epic 的 Niagara 概述](https://dev.epicgames.com/documentation/unreal-engine/overview-of-niagara-effects-for-unreal-engine) 与 [Simulation Cache 文档](https://dev.epicgames.com/documentation/en-us/unreal-engine/caching-and-reusing-your-system)。

## 资源和模拟的对应关系

母效果 `vfx_amb_welding_torch_sparks` 中有 6 个 EmitterEntityData。第 4 个局部对象的 Emitter 属性精确指向 `emt_amb_welding_sparks_falling` 的文件和导出实例 GUID。该发射器包含 19 个对象，四档质量均指向同一模板。沿 RootProcessor 和 NextProcessor 可走到 13 个处理器。其中 UpdateTransparency 和 UpdateColor 的 Pre 指针分别指向 PolynomialData 和 PolynomialColorInterpData。两者的输入枚举均为 2，即 `EfNormTime`。

| 源字段或连接 | UE 样本的处理 |
| --- | --- |
| SpawnRate 20，按时间分配，循环开关开启，模板周期 5 秒 | 每秒发射 20 个的连续 CPU 精灵发射器，使用 5 秒循环 |
| UpdateAge.Lifetime 0.5 秒，MaxCount 14 | 使用名义寿命 0.5 秒。实测稳态 10 个。14 是源上限，本样本没有触发该上限，也未实现独立的超限淘汰策略 |
| SpawnSize 0.05，随机求值器范围 0.3–1.1 | 按 1 源单位为 1 米换算，Niagara 两轴使用同一次随机采样，尺寸范围 1.5–5.5 cm |
| SpawnSpeed −1，Gravity 10 | 在独立预览中选取 UE Z 作为竖直轴，初速 −100 cm/s，加速度 −1,000 cm/s²。母效果局部变换单独记入 JSON |
| 色彩求值器含两组 HDR 端点和四个系数 | 按夹取到 [0,1] 的三次多项式混合三通道端点，保留大于 1 的线性颜色值 |
| 透明度求值器含四个系数、ScaleValue 和上下限 | 按归一化年龄求值、乘尺度、按源范围夹取，与 RGB 一起写回粒子 Color |

工具采用 `((a*t+b)*t+c)*t+d` 解释系数顺序，每 1/128 归一化寿命采一个键，写入可编辑 Niagara 曲线。验证端直接按原系数求值。该对照验证曲线写入后在 UE 的行为；Anthem 运行时代码的实际计算顺序仍缺原作帧对照。随机寿命比例 0.399、重力随机参数 0.242 已从 EBX 读出，样本使用名义值。原作的随机序列、分布规则、旋转及运动拉伸尚未校准。

图集输入追到 `vfx/textures/atlas/debris/vfx_dirt_debris`，原始 Chunk 有 349,520 字节。样本使用 Niagara 默认精灵材质显示粒子，原作图集通道、第二纹理、透明度裁剪、原始材质和曝光尚未复现。验收针对单个发射器的运动和 RGBA；完整焊接效果还包含其他 5 个发射器及灯光、声音。

## 单个下落火花的回读

重新启动 UE 5.8.4 后将系统放入独立 PIE 世界，以 1/60 秒步长推进 96 步，每 6 步读取一个 SimCache 帧。16 帧合计 135 条粒子观测，同一存活粒子可在多个帧出现。前六帧存活数为 1、3、5、7、9、10，随后稳定为 10。归一化年龄、颜色、速度、尺寸和寿命均从引擎读取。

实测透明度相对解析多项式的最大绝对误差约 0.00066。线性 HDR 颜色中误差最大的通道为 G，约 0.057；该通道的源端点约为 672.83 和 91.36。竖直速度相对名义匀加速解的最大误差约 0.12 cm/s。每次运行的随机尺寸和预览像素数可能变化，报告保留该次读数。截图为暂停在稳定发射阶段的实际 Niagara 渲染。

## 支持范围

`FXAudit` 核查了 `vfx/` 前缀下 18,113 个 EBX 名称与内容版本组合。其中 EffectBlueprint 为 3,050 个名称、3,172 个版本，ScalableEmitterDocument 为 11,654 个名称、12,103 个版本。这里的版本按名称和解包内容 SHA-256 去重。补丁专属的其他目录前缀未计入这些 EBX 数量。全目录的 52 条 MeshEmitterResource 对应 4 个名称、4 份不同内容。

焊接母效果已覆盖五份不同发射器、六个放置实例的 Niagara 建图与逐粒子回放。单独的下落火花验收与母效果共用一份源发射器。剧情雾气样本可导出 SplineData 的 X、Y、G 数组和 SplineType；已在 UE 验算的分段曲线限于焊接烟雾。其他发射器的自动转换、碰撞触发子发射、Mesh／Ribbon／Beam 动态绘制与 GPU 模拟仍待验证。
