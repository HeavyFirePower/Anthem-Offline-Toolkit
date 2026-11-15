# Anthem UI 与特效图形工具

本机样本在 `output/UI-FX-Preview`。`Previews` 可直接看 UE 输出的 PNG，`UE/AnthemUIFXPreview.uproject` 可以打开可编辑资产，`Assets/anthem-ui-fx.json` 可再次导入其他工程。进入工程后，在内容浏览器搜索 `WBP_Options_Brightness`、`WBP_G_PCButton`、`NS_Amb_WeldingTorch` 或 `NS_Amb_WeldingSparks_Falling`，双击打开。要看烟雾或耀斑，可以在 Niagara 中单独启用相应发射器，也可以看提高曝光后的分层 PNG。

## 按游戏路径导出

1. 双击 `Export-UI-FX.pyw`，选含 `catalog.sqlite` 和 `raw` 的解包目录。可按游戏路径、ID、UI、FX、UMG、Niagara、Font Face 或「仅源文件」搜索。
2. 多选资产，或点「按类别随机抽样」。完整清单下每次抽 7 项，覆盖字体、控件、特效、发射器、专用图集等。
3. 选输出父目录，点「导出选中项」。每次创建新批次。路径碰撞时在文件名追加版本 ID 前 16 位。两个游戏名称末尾的空格以 `_x20` 落盘，清单保留原名和来源 CAS。
4. 移动批次时保留完整目录。EBX 保存源字节，成功解析的对象另有 `.graph.json`。TTF 直接保存字体本体。可转换的 EBX 和专用图集旁有 `.ue` 目录，包含重建所用文件。清单记录 SHA-256。

| 本安装可浏览的资源 | 可以导出的内容 | 可导入 UE 的范围 |
| --- | --- | --- |
| `ui/` 下 7,171 个 EBX 路径与内容版本 | EBX、可解析的对象和布局连线 | PC 按钮和亮度校准界面的两个确定版本可生成 Widget Blueprint |
| 9 个字体版本 | 原始 TTF、名称、字形数和表校验 | Font Face，保存后对照字体文件字节摘要 |
| `vfx/` 下 18,113 个 EBX 路径与内容版本 | EBX、可解析的处理器、曲线和外部引用 | 焊接母效果与独立下落火花可生成 Niagara System |
| 4 个 MeshEmitterResource 版本 | 原始 RES | 动态网格粒子绘制待转换 |
| 77 个专用 AtlasTexture 版本 | RES、唯一匹配的 Chunk、可解析的网格及子区域坐标 | 有唯一规则网格且像素拼合通过时可导入首级 mip 的 Texture2D。清单记录降级原因 |

普通 Texture 资源使用 `Export-Textures.pyw`。选中的 UMG 和 Niagara 会随批次保存所需的 DDS 依赖。专用 AtlasTexture 使用独立的资源类型，源 Chunk 保留原始 mip 数据。

## 导入 UE 5.8.4

1. 双击 `Import-UI-FX-UE.pyw`，选择 `anthem-ui-fx.json`、目标 `.uproject` 和引擎安装目录。默认内容根为 `/Game/Anthem`。UE 包名里的空格、连字符等字符转为下划线。
2. 点「读取清单」，窗口列出可转换的行，并显示仅保留源文件的数量。选中需要的行，点「导入并保存选中项」。UMG 和 Niagara 在原游戏路径下建专属目录存放蓝图、系统及配套资产。独立字体和图集直接使用对应包路径。
3. 导入 UMG 时，工具会把匹配 UE 5.8.4 Win64 的 `AnthemUI` 插件及源码安装到目标工程的 `Plugins/AnthemUI`。迁移这两个控件时需一起带上插件，以便加载父类和处理输入。特效使用引擎内置 Niagara。遇到异源资产占用路径时工具报告冲突，重复导入同一批次会复用已保存资产并重新验收。
4. 每项保存后独立重开 UE。UMG 检查字体、纹理、布局与交互，并从渲染图取样。Niagara 在 PIE 中回放并读回粒子属性。独立图集使用线性 BGRA8 首级、关闭流送，核对尺寸、设置及像素摘要。逐项日志、预览和验收报告位于目标工程的 `Saved/AnthemImport`。

## 本轮样本与验收

种子 `20261115` 分层随机选 7 项，另加亮度界面和独立下落火花。9 份均已导出，6 份通过 UE 保存和重开，含依赖共生成 29 个资产包。样本的 `Reports/result.json` 记录 UE 包路径和对应回读报告。

| 样本 | 本次核对结果 |
| --- | --- |
| PC 按钮 | 普通、悬停、按下三帧背景 Alpha 分别为 102、204、51，鼠标在控件上释放触发一次 |
| 中英文亮度校准界面 | 14 个绑定与输入状态通过，覆盖键盘、手柄、确认、取消回滚和边界值。两个尺寸的三幅标识共 6 处像素与源图一致 |
| 焊接母效果 | 五份源模板、六处放置。12 个采样帧累计读回 1,565 条粒子观测，另验烟雾、耀斑和火花的分层图 |
| 独立下落火花 | 16 个采样帧、135 条粒子观测，检查颜色、透明度、尺寸和速度 |
| Zanzinger Light 字体、枪口火焰图集 | 字体字节摘要一致。图集为 1024 × 1024、2 × 2 帧，UE 首级像素摘要一致 |
| HUD 倍率控件、Wolven 电弧发射器、地下城雾 MeshEmitterResource | 保留 3 份原始资源，两份 EBX 另有解析图。尚无对应 UMG／Niagara 转换 |

亮度样本验证了界面模型值和输入响应。焊接使用 Ultra 模板和部分材质节点，原作场景照明、扭曲与同镜头外观仍待对照。[UMG 验收](UI字体与UMG.md) · [Niagara 验收](特效粒子曲线与Niagara.md)

## 命令行

原有 `Run-Anthem.ps1` 的 `UIFonts`、`UITrace`、`UISample`、`UIBrightness`、`FXTrace`、`FXSample`、`FXComposite` 可继续使用。以下命令与 GUI 共用批量处理逻辑。导出目的地须为一个尚未创建的目录。

```powershell
python tools/ui_fx_library.py --random --seed 20261115 --dest output/MyVisualSample
python tools/ui_fx_library.py --match g_pcbutton --dest output/MyButtons
python tools/ui_fx_import.py output/MyVisualSample/anthem-ui-fx.json "D:/UE/MyProject/MyProject.uproject"
```
