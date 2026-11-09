---
name: mujoco-gui
description: >-
  面向 MuJoCo Studio 和 MuJoCo viewer 插件的 Dear ImGui 与 ImPlot 界面设计
  （C++ ux/ 组件与 Python ViewerPlugin build_gui 面板）：即时模式控件布局、
  多行控件/列对齐（Tables API）、工具提示长度与宽高比、build_gui 与 render
  分离、ID 栈规则（## 标签消歧 vs ### 种子重置覆盖）、列表虚拟化
  （ImGuiListClipper）、主题化以及输入捕获（WantCaptureKeyboard）。仅在
  MuJoCo Studio、MuJoCo viewer 插件或 MuJoCo/机器人仿真与控制工具中构建、
  对齐、设置样式或调试 Dear ImGui 控件和 ImPlot 图表时使用。不要用于与
  MuJoCo 或机器人无关的通用 C++ 或 Python Dear ImGui / ImPlot /
  imgui_bundle 桌面应用，也不要用于 Studio 插件生命周期/消息传递（请使用
  mujoco-studio）或核心物理步进（请使用 mujoco-python）。
---

# Dear ImGui 开发与架构指南

> [!TIP]
>
> **MuJoCo 官方文档（`doc/*.rst`）及相关技能：**
>
> -   [`../../studio/python_plugins.rst`](../../studio/python_plugins.rst)
>     （`.. _StudioUiPlugins:`、`.. _StudioImGuiTips:`）和
>     [`../../studio/architecture.rst`](../../studio/architecture.rst)
>     （`.. _StudioCppComponents:`）——Dear ImGui 和 ImPlot 如何集成到
>     MuJoCo Studio、`UpdateEvent` 与 `BuildGuiEvent` 的区别，以及 C++（`ux/`）
>     与 Python 控件的取舍。本地存在这些简短 `.rst` 章节时直接阅读，或在
>     `pip install mujoco` 的 `pathlib.Path(mujoco.__file__).parent / 'doc'`
>     下查找，或访问
>     `https://mujoco.readthedocs.io/en/latest/<path>.html`（发布后可用
>     `/en/stable/`；Sphinx 锚点会将标签转为小写，例如
>     `#studioimguitips`）。
> -   [`../../programming/ui.rst`](../../programming/ui.rst)（`.. _UI:`）——
>     旧版 `mjui` C 框架，仅用于经典 `simulate.cc`（与 Studio 的 Dear ImGui
>     技术栈不同）。
> -   [mujoco-studio](../studio/SKILL.md) —— Studio viewer 启动器、解耦的
>     仿真/viewer 线程、插件、热重载以及 `WebViewer` 流式传输。

> [!NOTE]
>
> Dear ImGui 是即时模式 GUI 框架，专为快速迭代、最小状态开销和高响应性而设计。
> 本指南记录在 MuJoCo、其 Python 绑定以及 C++、Python 和 WebViewer 中的依赖目标
> 里编写可维护、高性能且可测试 UI 代码的最佳实践与约定。
>
> **同时适用于 C++ 和 Python**：所有架构准则、控件模式、Tables API 用法、
> ID 栈规则（`##` vs `###`）和性能原则，无论编写原生 C++（`ImGui::`）还是
> Python（`imgui.`）代码，都同样适用。虽然代码片段常用 Python 以保持简洁易读，
> 但对应的即时模式概念、控件 API、ID 栈机制、clipper 用法和样式规则可直接
> 1:1 对应到 C++。准则同时适用于核心 Dear ImGui 和扩展库，最重要的是 ImPlot。

## 1. 命名约定：`build_gui` vs `render` / `draw`

ImGui 代码不会直接向屏幕渲染像素。调用 ImGui 函数会构建并记录**绘制命令列表**
（顶点缓冲区、索引缓冲区、裁剪矩形和纹理引用），这些内容随后在帧流水线中
（通过 OpenGL、Vulkan 或 WebGL/WebViewer）被处理、光栅化并渲染。

-   **使用 `build_gui()` 或 `build_xxx_gui()`**（Python）或 `BuildGui()` /
    `BuildXxxGui()`（C++）命名所有提交 ImGui 控件的函数和方法（例如
    `build_control_panel_gui()`、`BuildControlPanelGui()`）。
-   **避免用 `render()` 或 `draw()`** 命名 UI 布局函数。`render` / `draw`
    严格保留给底层图形后端操作（如 `ImGui_ImplOpenGL3_RenderDrawData`）。

```python
# Good: Accurately describes building the frame's UI hierarchy
def build_gui(self) -> None:
  self.build_header_gui()
  self.build_table_gui()

# Avoid: Suggests immediate rasterization/rendering
def draw_ui(self):
  ...
```

## 2. GUI 与应用逻辑解耦

GUI 代码必须严格负责用户布局、呈现和输入转发。GUI 仅用于粘合底层逻辑，且该
粘合层应尽可能小、尽可能薄。切勿将业务逻辑、网络/RPC 查询、文件操作或仿真步进
直接混入帧构建函数。

-   **最小粘合层**：GUI 组件除将用户输入绑定到底层库调用并渲染当前状态外，
    不应做更多事情。
-   **独立库**：核心业务逻辑应放在独立的库或模块中（例如 `inspector_lib.py`
    与 `inspector_gui.py`）。
-   **异步执行**：昂贵操作（RPC、子进程、大型搜索）必须在工作线程/任务上运行，
    GUI 代码读取缓存状态或进度标志。
-   **可测试性**：解耦的库可通过标准单元测试充分测试，无需活动的 ImGui 上下文
    或窗口。

## 3. 自定义控件：即时模式纯函数

自定义控件应保持轻量、即时模式纯函数：

-   **纯即时模式**：输入作为参数传入，返回输出或变更元组（例如
    `changed, new_val = dial_widget(...)`）。避免重量级有状态框架抽象。
-   **避免控件级持久状态**：在控件内部跨帧存储状态会增加维护负担，并限制其在
    任意上下文中的复用。尽可能保持控件无状态；若状态必不可少，应存放在父应用
    状态中，而非控件内部。
-   **UX 定制与库**：当定制视觉控件（如旋钮、仪表）能提升 UX 时，鼓励使用。
    不要引入大型外部依赖，而是将小而专注的代码片段（必须为 MIT 许可）复制并
    重构为干净的即时模式函数。在 MuJoCo 中，共享的自定义 ImGui 控件功能位于
    Studio UX 模块的 `imgui_widgets.h`/`cc` 文件中。
-   **测试引擎就绪**：使用 Dear ImGui Test Engine 的钩子/宏暴露交互端点和
    边界框，以便自动化测试驱动能与自定义控件交互。
-   **实现前检查上游与变更日志**：在从头编写新自定义控件、布局行为或功能之前：
    -   查看 `docking` 分支的最新上游变更日志
        https://github.com/ocornut/imgui/blob/docking/docs/CHANGELOG.txt，
        并检查相对于 MuJoCo 中内置版本有哪些变化。
    -   搜索 Dear ImGui GitHub issues
        （https://github.com/ocornut/imgui/issues）和 Wiki
        （https://github.com/ocornut/imgui/wiki），查找现有解决方案、标准
        代码片段或相关讨论。
    -   **检查本地与上游（更新依赖）**：通常应同时检查本地检出的代码和
        GitHub 上的最新代码。如果该功能、控件或缺陷修复已存在于更新的上游版本
        中，更干净的解决方案往往是更新内置依赖，而不是在应用代码中维护自定义
        变通方案。

## 4. 禁用过时函数与内部头文件

ImGui 维护稳定的公共 API，并清晰记录弃用项：

-   **禁用过时函数**：在构建和框架更新时始终定义
    `IMGUI_DISABLE_OBSOLETE_FUNCTIONS`（以及
    `IMGUI_DISABLE_OBSOLETE_KEYIO`），防止弃用模式在代码库中残留。
-   **公共 API 稳定性**：依赖官方公共 API（`imgui.h`）。除非实现基础后端扩展，
    否则避免在应用和工具代码中包含内部头文件（`imgui_internal.h`）。

## 5. 性能与虚拟化：`ImGuiListClipper`

渲染可能包含大量条目的列表、表格或日志视图时（例如 100+ 个物体、仿真轨迹、
日志缓冲区）：

-   **始终使用 `ImGuiListClipper` / `imgui.ListClipper` 虚拟化**：不使用裁剪时，
    ImGui 会为列表中的每个条目生成顶点和文本布局，即使条目在屏幕外。在远程
    流式后端（`WebViewer`）中，未裁剪的表格会占满 WebSocket 流。C++ 中使用
    `ImGuiListClipper clipper; clipper.Begin(...)`；Python 中使用
    `clipper = imgui.ListClipper(); clipper.Begin(...)`。
-   **Clipper 用法**：clipper 计算可见范围，仅提交当前在视图中的条目。

```python
if imgui.BeginTable('ItemsTable', 1):
  clipper = imgui.ListClipper()
  clipper.Begin(len(items))
  while clipper.Step():
    for i in range(clipper.DisplayStart, clipper.DisplayEnd):
      imgui.TableNextRow()
      imgui.TableNextColumn()
      imgui.TextUnformatted(items[i].name)
  imgui.EndTable()
```

-   **内存与生命周期（`imgui.TextUnformatted`）**：显示大型或只读文本字符串时
    使用 `imgui.TextUnformatted`，避免 Python 绑定中的字符串复制和格式化开销。

## 6. 运行时 ID、ID 栈与测试自动化（`##` vs `###`）

Dear ImGui 通过将控件标签文本与当前 ID 栈种子进行哈希来标识控件
（`ImGuiWindow::GetID` 在 `imgui.cpp` 中调用 `ImHashStr`）。理解字符串运算符
`##` 和 `###` 对于避免 ID 冲突、在动态标签变化时保持状态以及编写可自动化 UI
至关重要。

### 技术机制：`##` vs `###`

两个运算符都依赖 Dear ImGui 内部的文本渲染和哈希例程：

-   **显示边界（`FindRenderedTextEnd`）**：文本渲染在首次出现 `##` 时停止。
    由于 `###` 也以 `##` 开头，**`##` 或 `###` 之后的任何字符都不会显示给
    用户**。
-   **隐藏消歧符（`##`）**：
    -   `ImHashStr` 哈希**整个字符串**（`"Label##Suffix"`），包括可见标签和
        隐藏后缀。
    -   **用途**：消歧同一 ID 作用域内共享相同可见标签的控件（例如多个
        "Delete" 按钮或重复的行条目）。
    -   **为什么 `Selectable(f'{body_name}##body_row_{i}')` 使用 `##`**：可见
        标签是 `body_name`，但列表可能包含重复的物体名称或重复条目。追加
        `##body_row_{i}` 会哈希完整字符串 `"{body_name}##body_row_{i}"`，
        生成唯一 ID，同时屏幕上仅显示 `body_name`。（或者将行渲染包裹在
        `imgui.PushID(i)` / `imgui.PopID()` 中）。若此处改用 `###`，
        `body_name` 将完全从哈希中丢弃。
-   **ID 覆盖 / 种子重置（`###`）**：
    -   `ImHashStr` 会专门检测三个 `#` 字符（`###`）。遇到时，它会**将哈希
        种子重置**回初始种子（`crc = seed`），并**仅**哈希 `###` 之后的子串。
        任何前置可见文本都完全从哈希中丢弃。
    -   **动态标签**：当可见文本跨帧变化时（例如
        `f'Items ({count})###ItemCounterBtn'` 或动画帧标题），`###` 保证控件
        内部 ID 保持不变。这保留了活动焦点、悬停状态、窗口停靠位置和折叠状态。
    -   **面向自动化的语义化名称**：对于纯图标按钮（例如
        `f'{ICON_FA_PLAY}###PlayButton'`，其中
        `ICON_FA_PLAY = '\uf04b'`），图标字形本身跨帧恒定，即使没有 `###`
        技术上也会产生稳定哈希。但若没有 `###PlayButton`，生成的 ID 来自不透明
        的 Unicode 码点。使用 `###PlayButton` 可赋予人类可读、语义明确的标识符，
        便于代理驱动的 Dear ImGui Test Engine UI 自动化和计算机控制按名称
        （`"PlayButton"`）可靠定位、检查和点击控件。

> [!NOTE]
> Dear ImGui Test Engine 目前尚未随 MuJoCo 捆绑，但计划支持，同时计划支持
> 录制的 UI 交互测试。

```python
# Icon glyph constant (e.g. FontAwesome play button unicode "\uf04b")
ICON_FA_PLAY = '\uf04b'

# Semantic ID override: The glyph is displayed, but the ID hashes only
# 'PlayButton'. Provides a meaningful identifier for test automation agents and
# tooling.
imgui.Button(f'{ICON_FA_PLAY}###PlayButton')

# Dynamic label with stable ID: Visible text changes with 'count', but the ID
# remains hashed from 'ItemCounterBtn', preserving focus, click, and hover
# state.
imgui.Button(f'Items ({count})###ItemCounterBtn')

# Disambiguated duplicate label: Visible text is 'body_name', but the ID hashes
# the full string '{body_name}##body_row_{i}' to prevent ID collisions across
# rows.
imgui.Selectable(f'{body_name}##body_row_{i}', is_selected)

# ID Scoping alternative: Using PushID / PopID for row loops instead of ##
# suffixes
imgui.PushID(i)
imgui.Selectable(body_name, is_selected)
imgui.PopID()
```

### 更多 ImGui 与 ImPlot 参考

查阅上游 Dear ImGui 和 ImPlot 源码及指南以获取权威细节和模式：

-   **Dear ImGui ID 栈与 API 指南**：`imgui.h`（约第 600 行）涵盖 ID 栈操作
    （`PushID`、`PopID`、`GetID`）和控件消歧规则。
-   **Dear ImGui 程序员指南与字符串哈希**：`imgui.cpp` 顶部包含完整的程序员
    指南；约第 2470–2515 行包含 `ImHashStr`，记录了 `###` 种子重置。
-   **Dear ImGui FAQ**：Dear ImGui [FAQ](https://github.com/ocornut/imgui/blob/master/docs/FAQ.md)
    包含专门章节：*"How can I have multiple widgets with the same label?"* 和
    *"Why is my widget not reacting when I click on it?"*。
-   **交互式 ImGui 演示窗口**：`imgui_demo.cpp` 提供每个控件、表格功能和
    clipper 模式的完整可运行示例。在应用中调用 `imgui.ShowDemoWindow()` 可
    交互式探索功能。
-   **ImPlot 头文件与 API 指南**：`implot.h` 记录所有绘图 API、标志、样式属性
    和上下文管理。
-   **交互式 ImPlot 演示窗口**：`implot_demo.cpp` 包含折线、散点、柱状、阴影
    图和实时滚动子图的丰富交互演示。调用 `implot.ShowDemoWindow()` 可查看实现。

## 7. UI 设计与美学：网格对齐（Tables API）、工具提示与主题化成本

高质量 ImGui 工具在即时模式简洁性与清晰视觉结构和一致样式之间取得平衡：

-   **网格对齐控件与表单行（优先使用 Tables API）**：
    -   将控件排列成网格、键值检查器面板、参数表或多行控制栏，且标签、下拉框或
        滑块必须垂直对齐并共享等宽时，**优先使用现代 Tables API**
        （`imgui.BeginTable`、`TableNextRow`、`TableNextColumn`）。
    -   避免旧式或脆弱的对齐技术，如 `imgui.Columns()`（已过时）、在变长文本
        标签后链式调用 `SameLine()`（不同标签宽度会使后续每个控件水平偏移）、
        用 `SetNextItemWidth(px)` 硬编码像素宽度，或插入虚拟间距。
    -   **表单与多控件对齐配方**：
        -   传入 `int(imgui.TableFlags.SizingStretchProp) | int(imgui.TableFlags.NoPadOuterX)`
            用于无边框布局表格，使外左右边缘与周围非表格控件齐平。
        -   将文本标签放在 `imgui.TableColumnFlags.WidthFixed` 列中（省略显式
            宽度，让 ImGui 自动适应所有行中最宽的标签），并在
            `TextUnformatted(...)` 前调用 `imgui.AlignTextToFramePadding()`，
            使标签基线与带框控件（`Combo`、`Slider`、`Input`）垂直居中。
        -   将控件（`Combo`、`SliderInt`、`SliderFloat`、`InputFloat`）放在
            `imgui.TableColumnFlags.WidthStretch` 列中，使用隐藏的 `##` 标签和
            `imgui.SetNextItemWidth(-1)`，使该列中每个控件从完全相同的 X 坐标
            开始并拉伸到完全相同的长度。如果滑块还有不同长度的尾部右侧标签
            （例如 `'Quality'` vs `'Overlay'`），将这些尾部标签放在各自的
            `WidthFixed` 列中——直接对带可见