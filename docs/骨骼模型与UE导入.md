# Anthem 骨骼模型与 UE 5.8.4 导入

这一步验证 Spacebear 基础模型。它有独立骨架、六级 LOD 和八骨影响顶点，从已有的全量解包目录即可重现。工具读取本地 MeshSet、几何 chunk 和 EBX。需要 Python 3.12、64 位 Windows 和 FBX SDK 2020.2 运行库。本机使用 UE 5.8 附带的 `libfbxsdk.dll`。安装在其他位置时，将环境变量 `ANTHEM_FBX_SDK_BIN` 指向该 DLL 所在的目录。

## 从游戏目录导出

在含有 `Run-Anthem.ps1` 的游戏目录运行。第一条命令只处理 Spacebear 的 LOD0。脚本会从相邻的 `Anthem-Extracted/anthem` 找到 `catalog.sqlite` 和 `raw`，终端最后一行给出产物目录。

```powershell
.\Run-Anthem.ps1 -Mode SkeletalMesh
.\Run-Anthem.ps1 -Mode SkeletalMesh -Lod 1
```

可用 `-Output` 指定已有解包目录，用 `-Destination` 指定 FBX 的输出根目录。默认输出为索引所在目录下的 `skeletal/<模型与骨架配对的 SHA-256>/lodN/`。每个 LOD 生成一份独立 FBX。

| 文件 | 内容 |
| --- | --- |
| `*_lodN.fbx` | 二进制 FBX。包含渲染分区、材质槽、骨架、BindPose、蒙皮簇和权重，顶点与关节位移均使用厘米 |
| `source-skin.json.gz` | 源单位米。每根骨骼的名称、父索引、局部绑定、模型空间绑定及其源逆矩阵。每个分区保存局部骨表和逐点的全局骨索引、原始字节权重 |
| `manifest.json` | 模型、骨架和几何来源 ID，配对依据，当前 LOD 的顶点、三角面、影响数分布，以及 Autodesk SDK 重新打开 FBX 后的检查结果 |
| `ue-import.json` | 另行运行 UE 验证后生成。含已导入 FBX 的 SHA-256、UE 包路径、骨名和父子关系差异、绑定矩阵分量误差及权重误差 |

## 此样本怎样找到骨架

Spacebear 角色模板 `sbr_wildlife_bear_miniboss_char_template` 的 EBX 导入表同时含有基础模型和 `animation/sbr/sbr_master_skeleton` 的文件 GUID 与对象 GUID。前者是 `SkinnedMeshAsset`，其 MeshSetResource 字段等于模型资源 RID `ab7d990ad0401995`。后者的 Skeleton 指针指向 `animation/sbr/sbr_skeleton` 的 `SkeletonAsset`。工具核对这些链接，并检查 MeshSet 声明的骨数与 SkeletonAsset 一致。

手动指定其他模型时，`AssetId` 填 MeshSet 的 Resource 记录 ID，`SkeletonId` 填 SkeletonAsset 的 EBX 记录 ID。二者均为 catalog.sqlite 中长度为 64 的十六进制字符串。加入 `OwnerId` 可检查共同持有者的导入表，有 MasterSkeleton 中转时再加 `MasterId`。仅指定前两个 ID 时，manifest 标为 `explicit_skeleton`，由使用者确认配对。相同骨数不足以识别骨架。

```powershell
.\Run-Anthem.ps1 -Mode SkeletalMesh `
  -AssetId 6820a5b7e495f241fa713aa944fa27f1328c19e96d59f031d6f533fda6a3b060 `
  -SkeletonId dcb1c75fc4a147c8efbb811599c7f9deaf202399b827844aa9f257795412614e `
  -OwnerId 83ef46b45b27ee275b3ea0c01211152373894e45e25f472b5aa1a00a52cc38bd `
  -MasterId 961a30e8f3e416a23e066775a7e72d739357067b8c960bd1860ec1ec8a56d486 `
  -Lod 0
```

用 SQLite 打开解包目录中的 `catalog.sqlite`。以下查询可用于按实际路径找到 ID。相同名称可能对应多个包、变体或补丁，结合 RID、内容哈希和引用判断。

```sql
SELECT id, kind, name, rid, sha256
FROM asset
WHERE status = 'ok'
  AND name IN ('enemies/creatures/sbr_spacebear/sbr_spacebear_model_mesh',
               'animation/sbr/sbr_skeleton',
               'animation/sbr/sbr_master_skeleton',
               'enemies/creatures/sbr_spacebear/sbr_wildlife_bear_miniboss_char_template')
ORDER BY name, kind, id;
```

## 骨架与权重还原

按 SkeletonAsset 的原始次序读取 BoneNames、Hierarchy、LocalPose、ModelPose 和 InverseModelPose。根为 `Reference`。在源行向量约定中，每个子骨的局部矩阵乘父骨的模型空间矩阵应得到自身模型空间矩阵。188 根骨骼的最大分量误差为 `4.12e-7`，模型空间矩阵乘源逆矩阵与单位矩阵的最大分量误差为 `3.28e-4`。

渲染分区使用各自的 bone palette。工具按顶点声明读取成对的 BoneIndices / BoneWeights 和可选的第二组，将分区局部索引映射到骨架，保留非零影响。每个控制点的权重字节和必须等于 255。Spacebear LOD0 中，148 根骨实际影响几何，其余骨仍保留在骨架中。1,244 个源顶点具有八个非零影响。FBX 中每个 cluster 的几何绑定矩阵为网格的静止变换，关联骨的绑定矩阵来自源 ModelPose。

文件写完后 SDK 重新打开 FBX 并检查每个骨名与父子关系、求值后的骨矩阵、绑定姿势与 cluster 矩阵、分区控制点与面数，以及每一条影响的控制点索引、骨名和权重。

| LOD | 渲染分区 | 源控制点 | 三角面 | 非零权重条目 | FBX 权重最大误差 |
| --- | ---: | ---: | ---: | ---: | ---: |
| 0 | 6 | 39,196 | 41,813 | 162,968 | 0 |
| 1 | 5 | 19,311 | 20,872 | 78,964 | 0 |
| 2 | 4 | 10,724 | 10,385 | 36,831 | 0 |
| 3 | 3 | 4,580 | 4,448 | 15,523 | 0 |
| 4 | 1 | 1,543 | 1,647 | 2,980 | 0 |
| 5 | 1 | 927 | 822 | 927 | 0 |

## 全量骨骼声明普查

对 1,233 份去重的 skinned MeshSet 检查了 7,091 个 LOD。遍历所有 LOD 的 12,529 个非空渲染分区，元数据解析错误为 0。有 9,395 个分区声明一组 `UShort4` 索引与 `UByte4N` 权重，3,134 个声明两组。分区声明的最大影响数为 1、2、4、6 或 8。全部已检查分区的骨表均无程序骨高位标志。该数字统计声明与骨表，尚未逐份核对对应的 SkeletonAsset、几何 chunk 或顶点实值。

使用以下命令复查。游戏目录中的相对输入路径为 `../Anthem-Extracted/anthem`。

```powershell
python tools/skeletal_census.py --out ../Anthem-Extracted/anthem --report ../Anthem-Extracted/anthem/skeletal-census.json
```

## 在 UE 中导入与验证

本次使用 UE 5.8.4 的传统 FBX 导入器。若在编辑器手动操作，在控制台执行 `Interchange.FeatureFlags.Import.FBX 0`，导入 LOD0 FBX，选 Skeletal Mesh，首次导入时让 UE 生成 Skeleton，启用 Convert Scene Unit，导入法线与切线，关闭导入动画、材质、纹理和自动创建 Physics Asset。生成的六个材质槽供后续绑定材质。LOD1 至 LOD5 已有独立 FBX，当前验证工程保存了 LOD0。

也可使用下面的命令，在工具自带的验证工程重复导入并保存。PowerShell 从游戏目录运行，`-EngineRoot` 可覆盖默认的本机引擎位置。

```powershell
$sample = .\Run-Anthem.ps1 -Mode SkeletalMesh
.\tools\Validate-SkeletalUE.ps1 -Manifest (Join-Path $sample 'manifest.json')
```

打开 `ue/AnthemValidation/AnthemValidation.uproject`，在内容浏览器进入 `/Game/AnthemValidation/SkeletalFBX`。打开 `SK_sbr_spacebear_model_mesh_lod0` 可查看模型、骨架树和材质槽。验证脚本核查同目录的 SkeletalMesh 与 Skeleton 两个资产已写入磁盘。

| UE 5.8.4 LOD0 回读 | 实测结果 |
| --- | --- |
| 骨名 / 父子关系 | 188 根骨，缺失、多出、父级不符均为 0 |
| 参考姿势 | 模型空间关节位置最大单分量误差 0.0000953 cm，旋转与缩放基向量最大分量误差 0.000000262 |
| 权重 | 对 MeshDescription 中对应的 39,196 个源点按骨名逐项比对，最大浮点误差 0.0000000297，量化回原始字节全部相同 |
| 顶点口径 | FBX 有 39,196 个控制点。UE MeshDescription 含 43,962 个点，构建后渲染网格有 37,152 个顶点。逐点权重结论针对可编辑 MeshDescription 中与源对应的点 |
| 渲染分区 / 材质槽 | 6 / 6 |

Spacebear 还通过了三段原作动画的回放与骨骼位置校验。角色和装备专题另覆盖 14 个 LOD0、程序骨默认锚点、Interceptor 换装及跨角色动作。打包后 GPU 蒙皮的逐顶点精度仍待验证。[动画解码与 UE 回放](动画解码与UE回放.md) · [角色配对、程序骨与模块化装备](角色配对程序骨与模块装备.md)

资源格式实现参考 Frosty 的 MeshSet 与 FBX 读取逻辑，UE 导入流程参照 Epic 的 Skeletal Mesh 文档。[Frosty MeshSet](https://github.com/CadeEvs/FrostyToolsuite/blob/1.0.6.3/Plugins/MeshSetPlugin/Resources/MeshSet.cs) · [Frosty 骨架与蒙皮实现](https://github.com/CadeEvs/FrostyToolsuite/blob/1.0.6.3/Plugins/MeshSetPlugin/FrostyMeshSetEditor.cs) · [Epic FBX Skeletal Mesh Pipeline](https://dev.epicgames.com/documentation/en-us/unreal-engine/fbx-skeletal-mesh-pipeline-in-unreal-engine)
