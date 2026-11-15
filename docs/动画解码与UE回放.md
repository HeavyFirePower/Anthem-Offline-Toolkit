# Anthem 动画解码、骨架关联与 UE 5.8.4 回放

本次验证使用 Spacebear 的三段连续动作。入睡和起身测试短片段的 8 位帧号，33 秒的睡眠循环测试 16 位帧号、内联常量和首尾姿势。三段都保留了原始关键帧、DOF 编号与通道名，已保存为使用同一个 Spacebear Skeleton 的 UE AnimSequence。

## 从已解包目录运行

在原来的游戏目录 `Games/Anthem` 打开 PowerShell。默认样本是 `SBR_AS_Sleeping_Ground_Enter`。第一条命令列出所选 AssetBank 中的片段名。第二条命令解码入睡片段，将它写入 UE 验证工程并回放。

```powershell
.\Run-Anthem.ps1 -Mode AnimationList
.\Run-Anthem.ps1 -Mode Animation -ImportUE
```

处理另外两段时可指定名称。每次只转换所选片段。省略 `-ImportUE` 会得到离线轨道和清单，适合先检查映射或批量选样。默认输出在相邻的 `Anthem-Extracted/anthem/animation` 目录，终端会打印每段的具体路径。

```powershell
.\Run-Anthem.ps1 -Mode Animation -Clip SBR_AS_Sleeping_Ground_Idle -ImportUE
.\Run-Anthem.ps1 -Mode Animation -Clip SBR_AS_Sleeping_Ground_Exit -ImportUE
```

要一次检查多段，可先解码并收集清单，再启动一次 UE 验证。

```powershell
$manifests = foreach ($action in 'Enter','Idle','Exit') {
    $folder = .\Run-Anthem.ps1 -Mode Animation -Clip "SBR_AS_Sleeping_Ground_$action"
    Join-Path $folder 'manifest.json'
}
.\tools\Validate-AnimationUE.ps1 -Manifest $manifests
```

打开工具附带的 `ue/AnthemValidation/AnthemValidation.uproject`，在内容浏览器进入 `/Game/AnthemValidation/Animation/2ac17e5916f99527df7fbd97`。双击名称含 `Sleeping_Ground_Enter`、`Idle` 或 `Exit` 的 AnimSequence，在动画编辑器中点播放或拖动时间轴即可查看。

工具包使用 Python 3.12 和本机 UE 5.8.4。新解压的工具包需要先用下面两条命令创建验证工程中的 Spacebear SkeletalMesh 和 Skeleton。本机游戏目录中的工程已有这两个资产。不同的 UE 安装位置可用 `-EngineRoot` 指定。

```powershell
$model = .\Run-Anthem.ps1 -Mode SkeletalMesh
.\tools\Validate-SkeletalUE.ps1 -Manifest (Join-Path $model 'manifest.json')
```

## 从片段追踪到骨架

| 来源 | Spacebear 的实际关联 |
| --- | --- |
| 动作 AssetBank | 选择同时含有控制器、压缩对象及 DOF 映射的 `ast_sbr_sleeping_ground_bundlegenbp_bundlegen_win32_antstate`，资源 ID 为 `97042fb6611ee26e1ae238ba95ece47fdec7f630d9cbe26b61f7ecad21159a1c` |
| 动作控制器 | `ClipControllerAsset.Anim` 是 64 位对象键。入睡片段指向 `121758039bc9390b` 的 EclipseAnimationAsset，基类 AnimationAsset 给出 ECLP 编码、末帧及 DOF 映射键 |
| 通道 | `ChannelToDofAsset.DofIds` 与压缩对象中的旋转、向量和标量通道顺序对应。使用 RigAsset 的 `RigDofSets`、`DofSetIdIndices`、`DofIds` 和各 LayoutAsset 的 Slots 解析实际名称 |
| 全局动画 Rig | 全局 AssetBank 的资源 ID 为 `ad0b159b1aefbc14014a248320cf12db2a08171133d4e32ef01fb55d4d19e072`。片段的布局键 `6fdbbcfd2583a3d9` 指向 `SBRAnimRig`，它位于 `SBR_skeleton` RigAsset 引用的布局层级中 |
| 原作模型骨架 | ANT SkeletonAsset 有 211 个关节。目标 EBX 的 188 根骨骼全部按名称和父级匹配。使用 ANT 默认局部位移与四元数计算模型空间参考位置，与 EBX 最大距离约 0.0002 厘米 |
| UE 目标骨架 | 创建序列前核对 Unreal 网格的全部骨名、层级、绑定位置及基向量。每个动画资产还引用模型已有的 Skeleton，避免生成重复骨架 |

同名路径下也有只含压缩对象的精简 AssetBank。要自行选择其他银行，使用 `-BankId` 指定完整的 Resource 记录 ID，先运行 `-Mode AnimationList`。可用 `-RigBankId`、`-RigName`、`-SkeletonId` 选择全局 Rig 与目标 EBX，使用 `-SkeletalMesh` 指定验证工程中的 UE 资产。不同角色需要重新通过布局引用、父级关系与绑定姿势核查。

## 压缩轨道与采样

ANT 银行的 GD.REFL 自带类型和字段布局，工具按反射读取各个 GD.DATA 对象。Eclipse 使用帧编号和变长关键帧序列。本次样本验证了以下表示。

| 数据 | 解码与去向 |
| --- | --- |
| 旋转 | 流式键占 6 字节，恢复三个量化分量和被省略的最大分量。单键使用描述符中的 64 位内联四元数。写入 UE 前使相邻四元数符号连续 |
| 位移 | 三个 16 位归一化值用通道 Min 和 Range 展开。两键内联形式直接保存两个时间和两个端点。源位移单位为米 |
| 标量 | 每键一个 8 位归一化值，使用本通道的 Min 和 Range 展开。睡眠样本包含脚部速度等辅助量 |
| 时间 | 多键帧编号按片段读取 8 位或大端 16 位整数。三段源 FPS 均为 30。保留控制器 TimeScale，资产按源采样帧率写入，控制器的运行时变速另行处理 |
| 重采样 | 在原始整数帧上生成 UE 骨骼轨道，旋转使用最短弧球面插值，位移使用线性插值。ANT 与模型骨架的个别局部轴不同，例如 AITrajectory。转换使用父子绑定基校准，再写为 UE 单位和轴向 |

只有能对应到目标骨骼的旋转和位移写入 AnimSequence。其余真实通道保存在 `source-keys.json.gz`，例如 IK 效应器、运动预测和脚部速度。它们的用途与具体名称可在该片段的 `auxiliary_slots` 中查看。未打键的骨骼使用目标参考姿势。

## 实测结果

三段动画均重新从磁盘载入。在独立的 UE 5.8.4 模拟世界里使用同一 Spacebear SkeletalMesh，以正常速度播放至末帧，并比对正在播放的网格骨骼与同一时刻的 UE 压缩姿势。

| 原作片段 | 帧范围，30 FPS | 时长 | 原始通道 / 写入骨骼通道 / UE 骨骼轨道 | UE 压缩姿势相对解码目标的最大误差¹ |
| --- | --- | ---: | ---: | --- |
| Sleeping_Ground_Enter | 0–98 | 3.267 秒 | 100 / 65 / 60 | 0.00257 cm，0.0975° |
| Sleeping_Ground_Idle | 0–1000 | 33.333 秒 | 90 / 58 / 53 | 0.00190 cm，0.0767° |
| Sleeping_Ground_Exit | 0–100 | 3.333 秒 | 100 / 65 / 60 | 0.00471 cm，0.1050° |

¹ 每段在首帧、第二帧、四分之一、中点、四分之三和末帧抽查所有已写入骨骼。位移取欧氏距离，旋转取四元数夹角。模拟回放另外按约 0.2 秒间隔检查模型空间位置，游戏目录中的交付工程回读最大距离误差为 0.00124 厘米。独立重复运行时，抽样时刻随编辑器帧间隔变化。比较基准是离线解码和 UE 姿势，原作运行时画面与插值器精度仍缺少直接对照。

独立检查了不同压缩片段之间的衔接。入睡末帧与循环首帧共有 76 条旋转和 13 条位移轨道，最大差异为 0.031° 与 0.031 厘米。循环末帧与起身首帧的相应差异为 0.0069° 与 0.0018 厘米。循环自身的位移首尾完全一致。比较涵盖流式四元数与内联四元数之间的交叉配对。

## 实际覆盖范围

1,668 条 AssetBank 目录记录按内容 SHA-256 去重得到 1,206 份，全部完成结构普查。723 条 AnimTrackData 对应 68 种不同内容，维持原始导出。部分 AnimTrackData 是特效属性曲线，角色动作从 ANT AssetBank 的控制器定位。

| 实际出现的动画类型 | 按对象键去重 | 验证进度 |
| --- | ---: | --- |
| EclipseAnimationAsset | 10,770 | 三段 Spacebear 通过轨道解码、骨架关联和 UE 回放 |
| RawAnimationAsset | 16 | 已识别对象 |
| FrameAnimationAsset | 17 | 已识别对象 |
| CurveAnimationAsset | 4 | 已识别对象 |
| DctAnimationAsset | 4 | 已识别对象 |
| VbrAnimationAsset | 1 | 已识别对象 |

Eclipse 的 `FloatFormat/VectorFormat/QuatBits/KeyTimeFormat` 组合中，`0/1/1/0` 有 9,040 个键，`0/1/1/1` 有 1,319 个键，`0/0/1/0` 有 411 个键。前两种组合各有实际回放样本，第三种尚待解码。其他角色、裁切与增量片段、根运动提取、控制器事件、运行时 IK 和状态机尚待验证。格式普查的分母与逐片段回放的分母分别记录。

## 文件与复查

| 文件 | 内容 |
| --- | --- |
| `manifest.json` | 源资源与对象键、骨架配对、帧率、时长和覆盖统计 |
| `source-keys.json.gz` | 全部已解码的源通道、原始帧号、DOF 名与压缩类型 |
| `target-bind.json.gz`、`ue-tracks.json.gz` | 目标参考姿势与实际写入序列的逐帧局部变换 |
| `ue-playback.json` | 保存的资产路径、数据哈希、帧数、原始与压缩姿势误差 |
| `ue-world-playback.json` | 独立进程重新载入的资产、播放时钟、终点状态与实际网格骨骼测量 |

工作区的全量普查和片段边界报告位于 `output/animations/census.json`、`output/animations/transitions.json`。在游戏目录中可执行以下命令重新统计。

```powershell
python tools/animation_census.py --out ../Anthem-Extracted/anthem --report ../Anthem-Extracted/anthem/animation-census.json
```

GD.REFL 与 GD.DATA 容器的字段组织参考公开的 IceBloc 实现。Eclipse 的通道编码、绑定校准、区间检查与上述数值均来自本机 Anthem 资源。UE 动画编辑器可使用时间轴检查这些已保存的序列。[IceBloc GenericData](https://github.com/marv7000/IceBloc/blob/master/IceBlocLib.Frostbite2013/GenericData.cs) · [Epic Animation Sequences](https://dev.epicgames.com/documentation/en-us/unreal-engine/animation-sequences-in-unreal-engine)
