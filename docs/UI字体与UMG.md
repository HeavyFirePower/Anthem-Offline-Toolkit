# Anthem UI 字体、布局与 UMG

## 从已解包目录复现

在含有 `Run-Anthem.ps1` 的工具根目录或游戏目录运行。入口会查找已有的 `catalog.sqlite` 和 `raw`。可用 `-Output '其他路径/anthem'` 指定解包目录，用 `-Destination '目标目录'` 改写本次结果位置。

```powershell
.\Run-Anthem.ps1 -Mode UIFonts
.\Run-Anthem.ps1 -Mode UIAudit
.\Run-Anthem.ps1 -Mode UITrace -Match 'ui/_common/widgets/pcbutton/g_pcbutton'
.\Run-Anthem.ps1 -Mode UISample -ImportUE
.\Run-Anthem.ps1 -Mode UIBrightness -ImportUE
```

`UIFonts` 导出原始 TTF。加 `-ImportUE` 会创建 Font Face，并在第二个编辑器进程读回。只需提取或研究源数据时，运行 `UISample` 即可得到按钮清单、字体和 DDS。UE 导入默认使用本机 UE 5.8 的安装目录，可用 `-EngineRoot` 改写。随附 Win64 编辑器模块在 UE 5.8.4 上编译，移植到其他引擎构建时可用插件源码重编译。

默认结果在解包目录的 `converted/ui` 下。第一个命令写入 `fonts/fonts.json` 和同目录下的 `fonts/*.ttf`；第二个写入 `audit/audit.json`。第三个按 EBX 资产 ID 在 `trace/` 下写 JSON。同名的 Data 和 Patch 有不同内容时会输出各个版本。也可以从普查清单复制完整 ID，使用 `-Mode UITrace -AssetId <ID>`。按钮命令写入 `sample/widget.json`、`sample/fonts.json`、纹理及两次 UE 验证报告。每次成功运行的 `ue-build.json` 与 `ue-reopen.json` 有相同的 `run` 标识。

## 字体与语言

目录中的 9 份 `UITtfFontFile` 对应 8 个逻辑名称。各资源从第 0 字节起即为 TrueType sfnt。工具按原样写出所有字节，核对目录 SHA-256、sfnt 整文件校验和、各表校验和，并记录 `name`、`maxp` 字形数及 `head.unitsPerEm`。通过对应 `UITtfAsset.FontResource` 的 64 位 RID 关联到字体家族、粗体和斜体标志。UE 第二次启动后从保存的 Font Face 读取字节，对照导出 TTF 的 SHA-1，9 份全部相同。

| 源逻辑名 | 已验证文件 | 字形数 |
| --- | ---: | ---: |
| zanzinger-lt / zanzinger-lt-it | 2 | 776 / 777 |
| zanzinger-rg / zanzinger-rg-it | 2 | 777 / 777 |
| armingti-b5 | 2 | Data 14,419，Patch 14,420 |
| arcrystalheigb-md | 1 | 7,977 |
| malgun | 1 | 13,904 |
| motoyaaporostd-w3 | 1 | 9,357 |

按钮的字体样式 `buttonlegend_el_14` 分别提供 HD 14 和 SD 7.5 两套字号。每套有 20 条语言／字体引用，保存在 `widget.json → font_definitions`。样本用 HD 英文常规字体 `zanzinger-lt` 创建 `F_ButtonLegend_EL_14`。重新打开后沿 TextBlock → Composite Font → Font Face 再次验证字节摘要。其他语言字体可以单独导入；随语言切换字族、回退和斜体选择仍待验证。字体文件结构和 UMG 字体资产说明见 [OpenType 规范](https://learn.microsoft.com/en-us/typography/opentype/spec/otff) 与 [Epic 的 UMG 字体文档](https://dev.epicgames.com/documentation/en-us/unreal-engine/using-fonts-with-umg-in-unreal-engine)。

## 布局与交互

`UIAudit` 按 `ui/` 前缀统计 `status=ok` 的 EBX。按名称分组有 6,290 个，同名的不同内容另计时有 7,171 个名称与内容版本组合，其中以 `UIWidgetBlueprint` 为根类的有 892 个名称、1,265 个版本。结构与连线读取通过 1,224 份，其中 1,058 份含已识别的布局元素，1,010 份含事件连线，这两个子集有重叠。其余 41 份的类尺寸或数组引用与解析器不匹配，失败的 ID 与原因在 `trace_errors`。清单含源 ID、内容哈希及是否有 UMG 转换。通用 `UITrace` 读取对象类型、局部对象索引、外部导入 GUID 对、已识别元素的布局与层链接，以及属性、事件和链接的端点。端口编号为源文件中的无符号哈希，跨资产调用仍需继续解析依赖。

按钮转换使用 `ui/_common/widgets/pcbutton/g_pcbutton` 的一个 Patch 版本，原始 EBX 有 17 个对象、22 条属性连接、3 条链接和 8 条事件连接。下表给出源数据中的声明值。矩形位置使用 AnchorOffset，位置单位映射为 Slate 布局单位。

| 元素 | 大小 | Anchor | Offset | UE 控件 |
| --- | --- | --- | --- | --- |
| Widget | 200 × 60 | — | — | SizeBox、层 Canvas |
| PC BtnLeft | 120 × 60 | 0, 0.5 | 5, 0 | Group Canvas |
| Bkg | 100 × 60 | 0, 0.5 | 16, −1 | Image |
| Text | 410 × 60 | 0, 0.5 | 21, −1 | 文本布局框与 TextBlock |
| 点击区 | 113 × 60 | 0, 0 | 1, −1 | 透明 Button |

容器保留背景、文字、命中区的源顺序。通过 EBX 文件 GUID 与实例 GUID 解析外部引用，再由 RID 定位 BC3 背景 `readyhex9slice`。其唯一 mip 为 56 × 48，保存后 UE 读回的 10,752 字节 BGRA8 与解码结果一致。九宫格横向边界取 0.3、0.7，纵向边界取 0、1，映射到 Slate Box 边距。原作的中心区尺寸策略、不同 DPI 的锚定效果、字形水平间距和完整文字特效栈仍需视觉校准。该样本显式 `HardDropShadow` 字形脚本中的黑色阴影偏移 (3,3) 已映射到 TextBlock。

| 事件 | 目标状态 | 背景 alpha | 额外输出 |
| --- | --- | ---: | --- |
| 初始、鼠标离开 | LeftNormal | 0.4 | 常态调色板文字 |
| 鼠标进入 | LeftHighlight | 0.8 | 高亮调色板文字 |
| 左键按下 | LeftPressed | 0.2 | 按下调色板文字 |
| 左键释放 | LeftHighlight | 0.8 | OnPrimaryReleased |

这三组值由 InterfaceDescriptor → FloatHub → 背景 Alpha 解析。文字 RGB 来自外部调色板、Vec3Hub 和文字 Color 输入，Data 与 Patch 调色板在用到的三项上一致。两段 MathEntity 的指令和输出连接给出文字宽度加 45（容器及点击区）、加 10（背景）的公式，高度为 60。样本原始文本为空。在 UMG 使用时，创建 `WBP_G_PCButton` 实例并调用 `Set Label` 设置实际文本，绑定 `On Primary Released` 执行自己的动作。可在 UI Only 模式添加到视口并启用鼠标指针。

## 验证和查看

打开工具目录下的 `ue/AnthemValidation/AnthemValidation.uproject`，在内容浏览器进入 `/Game/AnthemValidation/UI`。双击 `WBP_G_PCButton` 查看 Designer 中的控件树。在自己的 UE 工程使用这个样本时，先启用随附的 `AnthemUI` 插件，再迁移蓝图及其字体、纹理依赖；交互实现位于它的 Runtime 模块。

验证工具在第二个 UE 5.8.4 编辑器进程中重新加载资产，把鼠标进入、左键按下、左键释放和离开事件送到已绘制点击区的 Slate 控件。在每一步检查文字颜色与背景透明度，释放通知计数为 1。另外用两段不同长度的测试文本验证三区宽度随文字变化。正常、悬停、按下的离屏图分别为 `preview.png`、`hover.png`、`pressed.png`，图上的 “Ready” 是验证输入。三图在背景内部同一像素 (50,15) 的 alpha 字节分别为 102、204、51，外部 (0,0) 均透明。可在 `ue-reopen.json` 查看结果。

上述按钮测试覆盖该控件内部的鼠标事件。下面的亮度界面单独验证了键盘和手柄动作。

## 整屏样本 · 选项 → 亮度校准

在工具或游戏目录运行 `.\Run-Anthem.ps1 -Mode UIBrightness -ImportUE`。使用已有全量解包即可。省略 `-ImportUE` 可单独提取源布局、事件、字体、贴图和文本。默认输出目录为 `converted/ui/brightness`。运行 `.\Run-Anthem.ps1 -Mode UIAudit` 可刷新逐版本清单，其中 `umg` 标记对应的已实现转换。

本样本锁定 Patch 的 `ui/menu/options/options_brightnesssubscreen`，源 SHA-256 为 `1292fc8ccc6964ee786c477bcc8a1587cfa04200242a3bb3d31ee6df9850eee9`。界面有 31 个对象、9 条属性连接、6 条链接及 17 条事件连接。`brightness.json` 保存源 ID、依赖的文件和对象 GUID、动作端点、各控件的坐标、源字体与文本表 ID。

| 核查项 | 源数据和 UE 实现 |
| --- | --- |
| 布局 | 设计画布 3840 × 2160。三张标识各占 1024 × 512，中心横向偏移分别为 −1200、0、+1200。标题锚点为 (0.5, 0.2)，选项列表为 (0.5, 0.8)，按键提示位于底边。UMG 使用等比缩放并居中。 |
| 双向数据绑定 | 通过菜单数据、选项 getter 和 setter 找到同一字段 `DylanOption_Brightness`。其范围为 0 到 1，默认 0.5，配置 Step 为 0。选项行计算 `max(Step,(Max-Min)/100)`，本样本步长为 0.01。UE 模型向进度条及百分比文字广播变化，左右输入写回同一模型。 |
| 导航 | 该页面只有一项可调数据。键盘 ← / → 或手柄十字键左右调整；Enter 或手柄 A 确认；Esc 或手柄 B 返回。源 EBX 提供逻辑动作，以上物理键是验证工程的映射。 |
| 交互状态 | 打开时保存进入值。调整立即更新模型，按住确认或返回仍在编辑，释放后关闭。确认保留当前值，返回恢复进入值。重开使用宿主持有的模型。 |
| 文本与贴图 | Patch 英文及简体中文表提供标题、选项名、确认、取消与选项描述。前四项用于界面显示，描述保留在清单。源字体样式分别提供标题 21、选项名 18、值 16 的 HD 字号。三张源图各为 512 × 256，UE 读回检查顶级 mip 的 BGRA 字节与 sRGB 标志。 |

### 已在 UE 5.8.4 实测的结果

首个编辑器进程写入蓝图和依赖，第二个进程重开，并在取得焦点的 Slate 虚拟窗口中分发按下／释放事件，使用 D3D11 离屏绘制。`ue-reopen.json` 含 14 项状态记录、两种尺寸下 12 个实际绘制矩形，以及校准标识的像素读回。

| 操作 | 模型、滑杆与百分比文字的读回 |
| --- | --- |
| 载入 37%，按键盘右，再按手柄左 | 37% → 38% → 37% |
| 从外部将模型设为 73%，按住并松开 Esc | 按住时仍在编辑且显示 73%，松开后关闭并恢复 37% |
| 切到简体中文，手柄右调至 38%，按住并松开 A | 按住时显示 38%，松开后关闭；重开仍为 38% |
| 在 100% 向右调，在 0% 向左调 | 分别维持 100% 和 0%；手柄 B 返回恢复本次进入值 38% |
| 用键盘确认 48% | 释放 Enter 后关闭并保留 48% |

`brightness-en.png` 是 960 × 540 的键盘界面，三标识中心的横坐标为 180、480、780。`brightness-cn.png` 是 1280 × 800 的手柄界面，使用 1280 × 720 居中的设计区域，三中心位于 240、640、1040。两张图均检查标题、列表、图例和三张标识的绘制矩形，每边容差 1.1 像素。每张图另在三个标识上各取一个不透明像素与解码后的源纹理核对，左、中、右的 RGB 分别为 (3,3,3)、(11,11,11)、(49,51,49)。这些值描述 UI 校准图自身；画面亮度传递函数仍需独立证据。

### 在自己的 UMG 页面中试用

1. 打开随附的 `ue/AnthemValidation/AnthemValidation.uproject`，在内容浏览器打开 `/Game/AnthemValidation/UI/WBP_Options_Brightness` 查看布局。迁移到别的项目时一同迁移引用资产并启用 `AnthemUI` 插件。
2. 在宿主蓝图使用 **Construct Object from Class** 创建并持有 `AnthemBrightnessModel`，用 **Set Value** 注入当前设置。通过 **Create Widget** 创建 `WBP_Options_Brightness`，用 **Add to Viewport** 显示它，调用 **Open Calibration** 传入模型；`Use Chinese` 控制是否使用简体中文。
3. 对所属玩家控制器调用 **Set Input Mode UI Only**，将此界面设为焦点。绑定模型的 **On Changed** 可接收实时调整，绑定界面的 **On Closed** 可读取是否接受及最终值。宿主负责存档和返回上一级菜单。

按键提示的方括号文字、按钮外观、列表内居中与字形缩放取 UE 重建值。按钮提示支持鼠标点击，源界面图例的鼠标分支、音效和过渡时序仍待逐条对照。校准标识使用原始纹理，当前调整驱动 UI 模型；原作亮度传递函数、HDR、显示器响应、原作逐像素画面、物理设备与全视口输入链路、跨菜单栈导航以及其余语言回退仍待验证。
