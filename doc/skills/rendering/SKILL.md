---
name: mujoco-rendering
description: >-
  使用 Python 中的 mujoco.Renderer 和 MuJoCo C/C++ 渲染管线（mjrContext、mjvScene、mjr_render、
  mjr_readPixels）进行 MuJoCo 离屏图像渲染：RGB、度量深度图、分割掩码（mjtObj）、MJCF
  offwidth/offheight 帧缓冲尺寸、MUJOCO_GL（osmesa、egl）、MuJoCo
  相机（固定、跟踪、自由、MjvCamera）以及 mjvOption 标志/geomgroup。
  仅用于无头相机渲染、数据集录制和 MuJoCo 仿真中的视觉观测。不要用于通用 OpenGL/EGL/OSMesa
  离屏渲染、glReadPixels 深度/分割后处理，或不使用 `mujoco` 库的合成数据集流水线，也不要用于
  交互式查看器/Studio（mujoco-studio）或 ImGui 控件（mujoco-gui）。
---

# MuJoCo 渲染：相机、上下文与离屏缓冲区

> [!TIP]
>
> **官方 MuJoCo 文档（`doc/*.rst`）：** 在 MuJoCo 检出目录中时，直接阅读
> [`../../programming/rendering.rst`](../../programming/rendering.rst)
> （`.. _Rendering:`、`.. _FilamentRendering:`、`.. _ClassicRendering:`）和
> [`../../programming/visualization.rst`](../../programming/visualization.rst)
> （`.. _Visualization:`）；对于 [`../../python.rst`](../../python.rst)（`.. _PyRender:`）、
> [`../../XMLreference.rst`](../../XMLreference.rst) 和
> [`../../APIreference/functions.rst`](../../APIreference/functions.rst) 等大型参考文件，
> 用字面量 `.. _Label:` 锚点进行 grep。在检出目录之外，同一 `doc/*.rst` 树会离线随
> `pip install mujoco` 发布在
> `pathlib.Path(mujoco.__file__).parent / 'doc'`；否则获取
> `https://mujoco.readthedocs.io/en/latest/<path>.html`（已发布页面用 `/en/stable/`；
> `functions.rst` 渲染在 `APIreference/APIfunctions.html` 内，Sphinx 锚点会将标签转为小写，
> 例如 `.. _PyRender:` → `#pyrender`），并在
> `https://github.com/google-deepmind/mujoco/tree/main/` 下浏览仓库路径（`../../../`）。
>
> **相关技能：**
>
> -   [mujoco-python](../python/SKILL.md) — 核心物理仿真、步进和状态管理。
> -   [mujoco-studio](../studio/SKILL.md) — 交互式仿真、Studio 查看器应用和插件架构。
> -   [mujoco-gui](../gui/SKILL.md) — Dear ImGui UI 设计和自定义控件。

## 1. 概述：渲染后端与工作流

MuJoCo 提供两个内置 3D 渲染后端（在
[programming/rendering.rst](../../programming/rendering.rst) 的 `.. _Rendering:` 中并排比较）：

-   **经典 OpenGL（`mujoco.Renderer` / C `mjr`）**：立即模式 OpenGL 1.5
    渲染器（`mjv_updateScene` -> `mjr_render` -> `mjr_readPixels`），现处于
    维护模式。随 `pip install mujoco` 开箱即用，使
    `mujoco.Renderer` 成为无头 RGB、深度和分割捕获最简单的单类辅助工具。参见
    [programming/rendering.rst](../../programming/rendering.rst)
    （`.. _ClassicRendering:`）和 [python.rst](../../python.rst)
    （`.. _PyClassicRender:`）。
-   **Filament PBR（`mujoco.mjrf` 和 `mujoco.rendering.filament` / C `mjrf`）**：
    保留模式、异步基于物理的渲染器，支持 OpenGL、
    Vulkan、Metal 和 WebGL（拥有自己的无头或窗口化 GPU 上下文和
    任意大小的 `mjrfRenderTarget` 缓冲区）。推荐用于 PBR 材质、
    基于图像/光度照明以及新的 C++/Studio 应用；参见 §7
    和 [programming/rendering.rst](../../programming/rendering.rst)
    （`.. _FilamentRendering:`）。
-   **交互式查看器（`mujoco.studio` / `mujoco.viewer`）**：桌面或
    Web 流式交互式查看器；请改用 [`mujoco-studio`](../studio/SKILL.md)。

## 2. Linux 上的无头上下文：`MUJOCO_GL`

在无头 Linux 系统（服务器、容器）上，**在导入 `mujoco` 之前**通过 `MUJOCO_GL` 环境变量配置 OpenGL 上下文提供者：

```python
import os
os.environ['MUJOCO_GL'] = 'egl'      # Hardware-accelerated offscreen on NVIDIA GPUs
# os.environ['MUJOCO_GL'] = 'osmesa' # Portable CPU software rasterization

import mujoco  # Must import AFTER setting MUJOCO_GL
```

或从命令行：
```bash
MUJOCO_GL=egl python3 my_script.py
```

## 3. Python Renderer 类（RGB、深度与分割）

`mujoco.Renderer` 管理 GL 上下文创建、场景几何同步
和缓冲区回读。**始终将其用作上下文管理器**：

```python
import mujoco
import numpy as np

model = mujoco.MjModel.from_xml_path('scene.xml')
data = mujoco.MjData(model)

with mujoco.Renderer(model, height=480, width=640) as renderer:
  mujoco.mj_forward(model, data)

  # 1. RGB Rendering
  renderer.update_scene(data, camera='overhead_cam')
  rgb = renderer.render()  # Shape: (480, 640, 3), dtype: uint8

  # 2. Metric Depth Rendering (distance in meters)
  renderer.enable_depth_rendering()
  renderer.update_scene(data, camera='overhead_cam')
  depth = renderer.render()  # Shape: (480, 640), dtype: float32
  renderer.disable_depth_rendering()

  # 3. Semantic Segmentation Rendering
  renderer.enable_segmentation_rendering()
  renderer.update_scene(data, camera='overhead_cam')
  seg = renderer.render()  # Shape: (480, 640, 2), dtype: int32
  # seg[:, :, 0] = object ID (-1 for background)
  # seg[:, :, 1] = object type (mjtObj enum, e.g. mjOBJ_GEOM)
  renderer.disable_segmentation_rendering()
```

### 在仿真循环内渲染

创建 `Renderer` 会分配 GPU 帧缓冲。在 RL 或评估循环中，
**在循环外创建一次渲染器**：

```python
with mujoco.Renderer(model, height=84, width=84) as renderer:
  for step in range(1000):
    mujoco.mj_step(model, data)
    if step % render_interval == 0:
      renderer.update_scene(data, camera='agent_cam')
      # Call .copy() because the renderer's internal buffer is reused
      frame = renderer.render().copy()
```

## 4. C++ 离屏渲染管线（mjr 与 mjv）

在 C++ 中，使用 `mjvScene` 和 `mjrContext`
API 执行离屏渲染：

```cpp
#include <mujoco/mujoco.h>

// 1. Initialize visualization structures
mjvCamera cam;
mjv_defaultCamera(&cam);
cam.type = mjCAMERA_FIXED;
cam.fixedcamid = mj_name2id(m, mjOBJ_CAMERA, "overhead_cam");

mjvOption opt;
mjv_defaultOption(&opt);

mjvScene scn;
mjv_makeScene(m, &scn, 2000);  // Buffer up to 2000 geoms

// 2. Initialize offscreen rendering context
mjrContext con;
mjr_defaultContext(&con);
mjr_makeContext(m, &con, mjFONTSCALE_100);

// 3. Render frame
mjrRect viewport = {0, 0, 640, 480};
mjv_updateScene(m, d, &opt, nullptr, &cam, mjCAT_ALL, &scn);
mjr_render(viewport, &scn, &con);

// 4. Read pixels into user buffers
unsigned char rgb[640 * 480 * 3];
float depth[640 * 480];
mjr_readPixels(rgb, depth, viewport, &con);

// 5. Clean up
mjr_freeContext(&con);
mjv_freeScene(&scn);
```

## 5. 相机定义与放置

MuJoCo 相机沿其**负 Z 轴**方向观察：
-   **-Z**：观察前方方向
-   **+X**：右
-   **+Y**：上

### 在 MjSpec 中定义相机

```python
# Fixed camera pointing down at ground plane
cam = spec.worldbody.add_camera(
    name='overhead',
    pos=[0, 0, 3.0],
    quat=[0.707, 0.707, 0, 0],  # 90 deg rotation around X
    fovy=60,
)

# Camera attached to robot wrist
wrist_cam = wrist_body.add_camera(
    name='wrist_cam',
    pos=[0.05, 0, 0],
    xyaxes=[0, -1, 0, 0, 0, -1],
    fovy=75,
)
```

### 自由相机（`MjvCamera`）

用于自定义交互式或轨道视角：

```python
cam = mujoco.MjvCamera()
cam.type = mujoco.mjtCamera.mjCAMERA_FREE
cam.lookat[:] = [0, 0, 0.5]
cam.distance = 2.5
cam.azimuth = 90.0
cam.elevation = -20.0

renderer.update_scene(data, camera=cam)
```

## 6. 帧缓冲与场景选项

### 离屏帧缓冲分辨率

离屏渲染分辨率不能超过模型中预分配的尺寸。在编译前调整 spec 中的缓冲区上限：

```python
spec.visual.global_.offwidth = 1920
spec.visual.global_.offheight = 1080
model = spec.compile()
```

### 单一 `<visual>` 块规则

MuJoCo 仅使用 XML 文件中定义的**最后一个** `<visual>` 块。多个
visual 块会静默覆盖先前的设置：

```xml
<!-- ❌ WRONG — second visual block silently resets offwidth/offheight to default -->
<visual><global offwidth="1920" offheight="1080"/></visual>
<visual><headlight ambient="0.3 0.3 0.3"/></visual>

<!-- ✅ RIGHT — consolidate into one visual block -->
<visual>
  <global offwidth="1920" offheight="1080"/>
  <headlight ambient="0.3 0.3 0.3"/>
</visual>
```

### Geom 组可见性与可视化标志

通过 `MjvOption` 控制渲染层：

```python
opt = mujoco.MjvOption()
opt.geomgroup[:] = False   # Hide all groups
opt.geomgroup[0] = True    # Show primary geometry (default group)
opt.geomgroup[1] = True    # Show secondary visuals

# Toggle visual debugging flags
opt.flags[mujoco.mjtVisFlag.mjVIS_CONTACTFORCE] = True
opt.flags[mujoco.mjtVisFlag.mjVIS_TRANSPARENT] = True

renderer.update_scene(data, camera='overhead', scene_option=opt)
```

## 7. Filament 基于物理的渲染（`mjrf`）

MuJoCo 的 Filament 后端（C/C++ 中为 `mjrfilament.h`，Python 中为 `from mujoco import mjrf`
和 `mujoco.rendering.filament`）提供保留模式 PBR，支持
OpenGL、Vulkan、Metal 和 WebGL，任意 `mjrfRenderTarget` 尺寸
（独立于 `<global offwidth offheight/>`），以及异步
`mjrfReadRequest` 像素回读：

-   **Python（`ModelScene` + `Renderer`）**：创建 `mjrf.Context`，通过
    `mujoco.rendering.filament.model_scene.ModelScene` 一次性绑定
    模型灯光和可渲染对象，用 `ms.update(data)` 就地更新，并通过
    `mujoco.rendering.filament.renderer.Renderer` 渲染到 NumPy 目标缓冲区 — 完整示例见
    [python.rst](../../python.rst)（`.. _PyFilamentRender:`）。
-   **C/C++ 与 CMake**：使用 `-DMUJOCO_USE_FILAMENT=ON`（或
    `-DMUJOCO_USE_FILAMENT_MJR_COMPAT=ON` 以获得 `mjr_` 直接替换兼容性）构建，
    并参阅 [programming/rendering.rst](../../programming/rendering.rst)
    （`.. _reProgramming:`、`.. _reSupport:`）。
-   **光度光强陷阱**：如果*任何* `<light>` 具有正
    `intensity`，Filament 会将整个场景从经典头灯 +
    默认环境照明切换为光度模式（关闭头灯
    并使零强度灯光变暗）。对于 PBR 材质、纹理和
    照明创作，请阅读
    [programming/rendering.rst](../../programming/rendering.rst)
    （`.. _reAuthoring:`、`.. _reMaterials:`、`.. _reLighting:`）并检查
    `../../../test/render/testdata/` 中的参考 PBR/IBL 测试场景
    （`pbr.xml`、`lighting.xml`、`emissive_bloom.xml`、`reflect_pbr.xml`、
    `ibl.ktx`）和 `../../../model/mug/mug.xml`。

## 8. 常见陷阱与最佳实践

1.  **GL 上下文泄漏**：始终关闭 `mujoco.Renderer` 或使用 `with` 块。
    在循环中未释放渲染器会耗尽 GL 上下文。
2.  **销毁顺序**：在 C++ 中，必须在底层 OpenGL 上下文销毁之前调用
    `mjr_freeContext`（参见
    [programming/rendering.rst](../../programming/rendering.rst)，
    `.. _reContext:`）。
3.  **缓冲区复用**：`renderer.render()` 返回内部缓冲区视图。如果要将帧存储在列表中，请调用
    `.copy()`。
4.  **按名称查找相机**：传入 `camera='camera_name'` 或整数 ID。
    避免传入模型中不存在的原始字符串 ID。

## 9. 关键参考

### 文档

-   [渲染指南（Filament `mjrf` 与经典 OpenGL `mjr`）](../../programming/rendering.rst)
    （`.. _Rendering:`、`.. _FilamentRendering:`、`.. _ClassicRendering:`）
-   [抽象可视化（`mjvCamera`、`mjvOption`、`mjvScene`）](../../programming/visualization.rst)
    （`.. _Visualization:`、`.. _viCamera:`、`.. _viOption:`）
-   [Python 渲染绑定](../../python.rst)（`.. _PyRender:`、
    `.. _PyFilamentRender:`、`.. _PyClassicRender:`）
-   [MJCF `<visual>` 与 `<camera>` 参考](../../XMLreference.rst)
    （`.. _visual-global:`、`.. _body-camera:`）、
    [相机建模](../../modeling.rst)（`.. _CCamera:`）和
    [MuJoCo Menagerie](../../models.rst)（`.. _Menagerie:`）
-   [C 渲染 API 函数](../../APIreference/functions.rst)
    （`.. _Visualization-api:`、`.. _OpenGLrendering:`、
    `.. _FilamentRenderingApi:`）和
    [C++ 渲染示例](../../programming/samples.rst)（`.. _saRender:`）

### 源代码与模型示例

-   [C API 头文件](../../../include/mujoco/mujoco.h) 和
    [`mjrfilament.h`](../../../include/mujoco/mjrfilament.h)
-   [C++ 离屏与 Filament 示例](../../../sample/render.cc) 和
    [Filament PBR 测试场景](../../../test/render/testdata/pbr.xml)