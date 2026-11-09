---
name: mujoco-studio
description: >-
  在 Python 和 C++ 中构建并启动交互式 MuJoCo Studio 查看器与插件，涵盖解耦的仿真/查看器线程、Snapshot 与 Event 消息传递、生命周期事件、WebViewer 浏览器流式传输以及 Python 热重载。仅用于为 MuJoCo MjModel/MjData/MjSpec 仿真启动 MuJoCo Studio 查看器（mujoco_studio.launch_web、launch_native、launch_passive、LaunchStudio），编写 MuJoCo Studio Python ViewerPlugin/SimPlugin 或 C++ RegisterPlugin（mjPLUGIN_LIB_INIT）扩展，同步 MuJoCo 仿真/查看器状态（ViewerHandle.sync、@messages.handler），热重载插件/模型（HotReloadEvent），配置 WebViewer WASM 流式传输，或调试 Studio 日志。不适用于非 MuJoCo 仿真，或不使用 MuJoCo 的通用 Web 查看器/边车，Dear ImGui/ImPlot 布局（mujoco-gui），核心步进（mujoco-python），或无头渲染（mujoco-rendering）。
---

# MuJoCo Studio：交互式仿真、UI 与插件架构

Studio 取代了旧版 `simulate` 工具，提供模块化交互式可视化技术栈：UI 基于 [Dear ImGui](https://github.com/ocornut/imgui) 和 ImPlot，渲染基于 Google Filament（PBR，支持 Vulkan、OpenGL、Metal 和 WebGL 后端），并通过 `WebViewer`（WASM）开箱即用地支持浏览器流式传输。其默认界面 **Studio Viewer App** 围绕可组合的 **Studio 插件** 设计，团队可跨环境编写（如 Gymnasium 或机器人套件）、资产检查/加载工具以及智能体监控/控制工作流共享这些插件。

> [!TIP]
>
> **官方 MuJoCo 文档（`doc/*.rst`）：** 详细的 Studio 架构、生命周期表和 C++/Web 文档位于
> [`../../studio/`](../../studio/index.rst)——本地存在时直接阅读这些简短的 `.rst` 章节，或在 `pip install mujoco` 的
> `pathlib.Path(mujoco.__file__).parent / 'doc'` 下查找（对 `XMLreference.rst` 等大文件保留 `.. _Label:` grep），或获取
> `https://mujoco.readthedocs.io/en/latest/studio/<page>.html`（发布后可用 `/en/stable/`；Sphinx 锚点会将标签转为小写，如 `#studiodiffsync`）。
>
> **相关技能：**
>
> -   [mujoco-gui](../gui/SKILL.md) — Dear ImGui 与 ImPlot 控件布局、Tables API 对齐、`##` 与 `###` ID 以及 `ImGuiListClipper`。
> -   [mujoco-python](../python/SKILL.md) — 核心仿真生命周期（`MjSpec` -> `MjModel` -> `MjData`）、命名访问和空间数学。
> -   [mujoco-spec-editing](../spec_editing/SKILL.md) — 程序化 `MjSpec` 建模、附件和运行时重新编译。
> -   [mujoco-rendering](../rendering/SKILL.md) — 无头离屏 RGB、深度和分割渲染。

## 官方 Studio 文档（按需加载）

此处不重复参考表，当任务涉及以下领域时，请阅读对应的 `.rst` 章节：

-   **[studio/index.rst](../../studio/index.rst)**（`.. _Studio:`、`.. _StudioQuickStart:`、`.. _StudioComparison:`）— 概述、CLI 快速入门，以及 `NativeViewer`、`WebViewer`、C++ Studio、MuJoCo Live 和旧版 `simulate` / `mujoco.viewer` 的功能对比。
-   **[studio/architecture.rst](../../studio/architecture.rst)**（`.. _StudioThreads:`、`.. _StudioCppComponents:`、`.. _StudioMessaging:`、`.. _StudioSnapshotsVsEvents:`）— Python 与 C++ 中的仿真/查看器线程拆分，何时在 C++（`ux/`）而非 Python 中实现重型或可移植到 WASM 的控件（GIL 释放、MuJoCo Live 与 `WebViewer`），以及 `Snapshot` 与 `Event` 通道语义。
-   **[studio/native_viewers.rst](../../studio/native_viewers.rst)**（`.. _StudioApp:`、`.. _StudioPythonNative:`、`.. _StudioLaunchers:`、`.. _StudioExecutionModes:`、`.. _StudioBackends:`）— 预构建独立桌面应用（macOS 上为 `MuJoCoStudio.app`，Windows 上为 `bin\mujoco_studio.exe`，Linux 上为 `bin/mujoco_studio`）与 Python `NativeViewer`（以及何时将共享自定义组件放在绑定到 Python 的 C++ 中）、`launch_native` / `launch_web` / `launch_passive`、阻塞式 `.run(...)` 与非阻塞式 `.launch(...)`，以及 `ViewerConfig.gfx` 模式（`opengl`、`vulkan`、`opengl_headless`、`opengl_software`、`vulkan_software`、`web`）。
-   **[studio/web_viewers.rst](../../studio/web_viewers.rst)**（`.. _StudioLive:`、`.. _StudioWebViewer:`、`.. _StudioBrowserSandboxing:`）— MuJoCo Live（`?model=github:...`）、`WebViewer` 的 `/ui` + `/model` + `/state` + `/drop` 流、Controller 与 Spectator 角色、COOP/COEP + `localhost`/HTTPS `SharedArrayBuffer` 规则（`ssh -L 8080:localhost:8080`），以及跨 WASM 和连续 Python 会话的持久 C++ 全局/静态状态。
-   **[studio/python_plugins.rst](../../studio/python_plugins.rst)**（`.. _StudioViewerPlugins:`、`.. _StudioSceneOverlays:`、`.. _StudioImGuiTips:`、`.. _StudioSplitPlugins:`、`.. _StudioLifecycle:`、`.. _StudioDiffSync:`、`.. _StudioPriorities:`、`.. _StudioHotReload:`）— 编写 `ViewerPlugin` 与 `SimPlugin` 扩展、`viewer.extra_geoms`（`MjvGeom`）与 `mujoco.mjrf`、完整的 Viewer 与 Sim 生命周期事件表（包括 `HotReloadEvent`）、`send_if_changes` 差异同步、处理器优先级以及 `--hot_reload`。
-   **[studio/cpp.rst](../../studio/cpp.rst)**（`.. _StudioCppLauncher:`、`.. _StudioCppPlugins:`、`.. _StudioLiveCpp:`）— C++ `LaunchStudio`、`LauncherConfig`、`GuiPlugin` / `ScenePlugin` / `ModelPlugin` / `KeyHandlerPlugin` / `SpecEditorPlugin` 回调、`mjPLUGIN_LIB_INIT` 与显式 `RegisterPlugin`，以及 Emscripten WASM 构建。

## 1. 架构与线程模型

### Studio 与旧版 `simulate`

| 功能 | 旧版 `simulate` / `mujoco.viewer` | Studio（`mujoco.studio`） |
| :--- | :--- | :--- |
| **GUI 与绘图** | 固定的 `mjui` 面板与 `mjr_figure` | **Dear ImGui** 与 **ImPlot** |
| **渲染后端** | 经典 OpenGL（`mjrContext`） | **Filament** PBR（Vulkan、OpenGL、Metal、WebGL） |
| **Web 与远程** | 无 | **WebViewer**（通过 HTTP/WS 流式传输 ImGui UI 与状态）与 **MuJoCo Live**（WASM） |
| **可扩展性** | 硬编码面板与快捷键 | Python 与 C++ 中的**插件架构** |
| **线程与同步** | 共享 `model`/`data` 并加锁 | Python 与桌面 C++：解耦的仿真/查看器线程；WASM（`MuJoCo Live`）：单线程 |

### 解耦的仿真与查看器线程（Python）

`viewer_protocol.py` 和 `viewer_handle.py` 通过非阻塞消息端点将仿真执行与渲染和 UI 解耦：

```text
┌────────────────────────────────────────────────────────┐
│              Simulation Thread (Caller)                │
│  - Owns MjModel and MjData                             │
│  - Runs physics loop, RL policies, or MPC planners     │
│  - Executes SimPlugin instances (e.g. StepControl)     │
│  - Drives simulation via handle.sync()                 │
└────────────────────────────────────────────────────────┘
                        ▲
                        │   Queued Events & Snapshots
                        │ (SimEndpoint <-> ViewerEndpoint)
                        ▼
┌────────────────────────────────────────────────────────┐
│              Viewer Thread (or Web Client)             │
│  - Owns Window, Renderer, Camera, VisOptions, Perturb  │
│  - Owns local copy of MjModel and MjData for rendering │
│  - Executes ViewerPlugin instances (studio_app, UI)    │
│  - Dispatches lifecycle events (UpdateEvent, BuildGuiEvent) │
└────────────────────────────────────────────────────────┘
```

-   **基于优先级的插件系统（`plugin.py`、`plugin_registry.py`）**：
    `plugin_registry.PluginRegistry` 发现 `@messages.handler` 方法并按优先级顺序分发消息。启动器接受单个
    `plugins=[...]` 列表，并按基类划分实例：
    -   **`plugin.SimPlugin`**：在仿真线程上运行，在 `SimInitEvent` 时自动缓存 `self.handle`，处理物理步进、重置、扰动、模型重载和策略更新。
    -   **`plugin.ViewerPlugin`**：在查看器线程上运行，在 `ViewerInitEvent` 时自动缓存 `self.viewer`，处理视觉更新、渲染叠加层和 Dear ImGui 构建。
-   **规范示例（`python/mujoco/sample/studio/`）**：
    -   `python/mujoco/studio/viewer.py` — 标准 Studio 查看器
        （`studio_app.App`）；使用
        `viewer.py --model=<file> --plugins=a.py,b.py` 运行（实例化这些文件中定义的每个
        `ViewerPlugin`/`SimPlugin`）。
    -   `python/mujoco/sample/studio/ghost.py` — `GhostRenderer`，在
        `UpdateEvent` 时注入瞬态 `extra_geoms`，在 `BuildGuiEvent` 时注入 ImGui 控件
        （`viewer.py --model=... --plugins=python/mujoco/sample/studio/ghost.py`）。
    -   `python/mujoco/sample/studio/implot.py` — `BodyInspector`，演示实时 ImGui +
        ImPlot 图表（`viewer.py --model=... --plugins=python/mujoco/sample/studio/implot.py`）。
    -   `python/mujoco/sample/studio/custom_sim_loop.py` — 外部脚本，通过 `handle.sync(model, data)` 拥有自己的步进和
        节奏循环。
    -   `python/mujoco/sample/studio/python_scene.py` — `MyScene` 和
        `MySceneGui`，演示用 Python 构建的可热重载场景与 UI 配对
        （使用 `--plugins=python/mujoco/sample/studio/python_scene.py` 运行 `viewer.py --hot_reload=all`）。

## 2. 启动 Python 查看器

Studio 提供三个运行器模块，共享相同的 `.run(...)` 和
`.launch(...)` 接口：

1.  **`launch_web.run`**（`python/mujoco/studio/launch_web.py`）—
    **推荐的默认选项。** 在后台守护线程中启动 `WebViewer`，**零主机图形依赖**（无需 X11/Wayland 显示服务器或
    主机 Filament 库）：物理和无头 Dear ImGui 布局
    在主机上运行，而 UI 绘制命令和状态通过
    WebSocket 流式传输到浏览器 WASM 客户端。非常适合远程工作站、
    容器和无头服务器。
2.  **`launch_native.run`**
    （`python/mujoco/studio/launch_native.py`）— 打开本地
    桌面窗口，使用原生 Filament 图形（Vulkan 或 OpenGL）。
3.  **`launch_passive.run`**
    （`python/mujoco/studio/launch_passive.py`）— 检查
    `config.gfx` 并委托给 `launch_web`
    （`config.gfx in ('web', 'webgl')`）或 `launch_native`。除非需要通过 `config.gfx` 在运行时切换，否则直接使用 `launch_web` 或
    `launch_native`。

### 阻塞式运行器（`.run`）

接受 `model_path=...`（由 `ViewerHandle` 通过
`LoadModelEvent` 在仿真线程上加载，设置 `handle.model_path` 以便 **Reload** 可用）或内存中的 `model=...`（可选 `data=...`，通过
`handle.set_model` 发布）。`studio_app.SIM_PLUGINS` 捆绑了
`step_control.StepControl()` 和 `sim_history.SimHistory()`：

```python
from mujoco.studio import app as studio_app
from mujoco.studio import launch_web  # Or launch_native / launch_passive
from mujoco.studio import viewer_protocol

config = viewer_protocol.ViewerConfig(title='Studio')

launch_web.run(
    config,
    model_path=model_path,  # Or model=model, data=data for an in-memory model.
    plugins=[
        studio_app.App(),
        *studio_app.SIM_PLUGINS,
    ],
)
```

> [!WARNING]
>
> **步进和节奏控制是插件的职责。** `studio_app.SIM_PLUGINS`
> 包含 `step_control.StepControl()`（实时节奏的 CPU 步进）和
> `sim_history.SimHistory()`（历史记录回溯）。没有 `StepControl()`（或
> 你自己的 `StepEvent` 插件），`handle.sync()` **不会**推进物理或
> 休眠——紧凑的 `while handle.is_running(): handle.sync(...)` 循环会以
> 100% CPU 空转。当替换 `StepControl()` 时（例如，用于 GPU rollout、自定义 RL
> 循环，或通过 `NullSimulation` 插件实现纯非物理网格/资产查看器，
> 该插件在每次迭代中节流而不调用 `mj_step`），请传入单独的
> 插件并显式管理节奏。

### 非阻塞启动器（`launch_web.launch`、`launch_native.launch`、`launch_passive.launch`）

仅当将查看器附加到外部循环且不希望将其反转
为 `StepEvent` 插件时（例如，现有的 RL 训练
循环、轨迹回放脚本或测试），才使用 `.launch(...)`：

```python
with launch_web.launch(
    config,
    plugins=[
        studio_app.App(),
        *studio_app.SIM_PLUGINS,
    ],
) as handle:
  while handle.is_running():
    # Mutate model.opt or data in step-event handlers, NOT here (see §5.2).
    model, data = handle.sync(model, data)
```

若要在调用者循环中直接步进和调节物理节奏，请省略
`StepControl()`，并在 `model, data = handle.sync(model, data)` 之后步进（参见
`python/mujoco/sample/studio/custom_sim_loop.py`）。

## 3. 插件与消息传递要点

`messages.py` 中所有跨线程通信使用两个通道基类
（自定义类按约定以 `*Snapshot` 或 `*Event` 结尾）：

-   **`Snapshot`（有损，最新优先）**：替换同类型任何未读的先前值；
    用于连续状态或完整控制配置，其中只有最新值重要（例如，`StateSnapshot`、`StepControlSnapshot`、
    `MjOptionSnapshot`）。
-   **`Event`（无损 FIFO 队列）**：保证按序、恰好一次
    投递；用于离散操作、重载和不能相互覆盖的独立编辑（例如，`ModelEvent`、`StateEvent`、`ResetEvent`、
    `SingleStepEvent`、`RequestPauseEvent`）。

插件通过 `@messages.handler` 订阅消息：

```python
from mujoco.studio import messages
from mujoco.studio import plugin

class MyPlugin(plugin.ViewerPlugin):
  @messages.handler
  def on_build_gui(self, event: messages.BuildGuiEvent) -> bool | None:
    # Build Dear ImGui widgets here (self.viewer is cached automatically).
    return False  # Return True ONLY to consume and block lower-priority handlers.
```

-   **优先使用 `Pre*`/`Post*` 生命周期事件而非 `messages.Priority`**：在匹配你意图的阶段事件上使用
    裸 `@messages.handler`，而不是在 `ModelEvent` 或 `StepEvent` 上调整优先级：
    -   **模型加载/重载后**：订阅 `PostModelEvent`（此时 `model`
        和 `data` 已安装且 `mj_forward` 已运行）。
    -   **仿真步进阶段**：`PreStepEvent`（写入控制、施加力或
        调用 `handle.set_model`）、`StepEvent`（