# MuJoCo Agent Skills

本目录包含面向 AI 编码代理和开发者的工作流指南与防护规则，覆盖 C++ 和 Python 下的 MuJoCo 使用。

## 官方文档（`doc/*.rst`）

这些技能侧重简洁的代码骨架、惯用模式和常见陷阱（`# Good:` 与 `# Avoid:`）。它们补充而非重复 `../index.rst` 中的官方 MuJoCo Sphinx 手册：

- **本地 `.rst` 文件 + Sphinx 锚点**：每个技能中的相对链接（如 `../../modeling.rst` 配合 `` `.. _CBodyOrdering:` ``）直接指向本 `doc/` 目录树。短 `.rst` 章节（如 `../studio/*.rst`）可直接阅读；对于大型参考文件，如 `../XMLreference.rst`（例如 `` `.. _sensor-contact:` ``）或 `../APIreference/functions.rst`（例如 `` `.. _mj_step:` ``），grep 字面量 `.. _Label:` 字符串即可跳转到目标章节。
- **独立安装（pip 包与在线回退）**：`mujoco` PyPI 包（`pip install mujoco`）在 `mujoco/doc/` 下打包了 `doc/**/*.rst` 和 `doc/skills/`（`pathlib.Path(mujoco.__file__).parent / 'doc'`），因此相对 `../../<path>.rst` 链接在那里也能离线解析。当本地没有 `doc/` 目录树时，每个 `doc/<path>.rst` 页面都发布在 https://mujoco.readthedocs.io/en/stable/ 上，对应 `<path>.html`（`main` 和未发布章节如 `studio/` 则位于 `/en/latest/`；`APIreference/functions.rst` 渲染在 `APIreference/APIfunctions.html` 内，Sphinx 锚点会将标签转为小写 `#<lowercase-label>`，例如 `.. _StudioDiffSync:` → `#studiodiffsync`），仓库相对路径（`../../model/`、`../../test/`、`../../sample/`、`python/*.ipynb`）位于 https://github.com/google-deepmind/mujoco/tree/main/ 下。
- **核心手册章节**：`../overview.rst` | `../modeling.rst` | `../XMLreference.rst` | `../models.rst`（`.. _Menagerie:`）| `../computation/index.rst` | `../computation/speculative.rst` | `../computation/fluid.rst` | `../programming/index.rst` | `../programming/samples.rst` | `../programming/extension.rst` | `../OpenUSD/index.rst` | `../python.rst` | `../mjx.rst` | `../mjwarp/index.rst` | `../studio/index.rst` | `../APIreference/index.rst`。

## 参考模型、测试夹具与示例

在自行编写合成 XML 或猜测专用特性的语法之前，先查看 MuJoCo 自带的规范模型和示例：

- **展示用 MJCF 模型（`../../model/`）**：自包含模型，覆盖关节机器人与机构（`humanoid/`、`car/`、`tendon_arm/`、`slider_crank/`、`mug/`）以及专用物理特性（`flex/`、`replicate/`、`plugin/`、`tactile/`、`adhesion/`、`sleep/`、`speculative/`、`cards/`、`cube/`、`hammock/`、`balloons/`）。
- **MuJoCo Menagerie（`../models.rst`，`` `.. _Menagerie:` ``）**：精选的即用型机器人描述（Franka Panda/FR3、UR5e/UR10e、Unitree G1/H1/Go2、Spot、Barkour、Shadow Hand、Robotiq 2F-85、ALOHA 2、Skydio X2、Flybody），位于 https://github.com/google-deepmind/mujoco_menagerie。
- **最小特性与单元测试 XML（`../../test/**/testdata/`）**：紧凑的单特性夹具，覆盖引擎物理（`test/engine/testdata/`）、`MjSpec`/编译器（`test/user/testdata/`）、XML 附件与 OpenUSD（`test/xml/testdata/`）、Filament PBR 材质与光照（`test/render/testdata/`）以及多体基准（`test/benchmark/testdata/`）。
- **C++ 与 Python 示例**：`../../sample/*.cc`（`../programming/samples.rst`，`` `.. _Sample:` ``）和 `../../python/mujoco/sample/`。
- **交互式 Colab / Jupyter notebook**：`python/tutorial.ipynb`（`` `.. _PyNotebook:` ``）、`python/mjspec.ipynb`（`MjSpec` 与 `PyMJCF` 对比）、`python/rollout.ipynb`（`mujoco.rollout`）、`python/LQR.ipynb`（`mjd_transitionFD` 与 LQR）、`python/least_squares.ipynb`（`mujoco.minimize.least_squares`）、`python/mujoco/sysid/sysid.ipynb`（`mujoco.sysid`，`` `.. _PySysid:` ``）以及 `mjx/tutorial.ipynb`（`` `.. _MjxNotebook:` ``）。
- **RL、控制与仿真生态**：
  - **MuJoCo Playground**（https://github.com/google-deepmind/mujoco_playground）—— 基于 JAX/MJX（`MJX-JAX` 与 `MJX-Warp`）的 RL 环境，覆盖运动、操作、灵巧手和视觉。
  - **mjlab**（https://github.com/mujocolab/mjlab）—— Isaac Lab 风格的模块化管理器 API，可直接用 `mujoco_warp` 和 PyTorch 训练策略（见 `../mjwarp/index.rst`，`` `.. _mjwFAQ:` ``）。
  - **MuJoCo Warp**（https://github.com/google-deepmind/mujoco_warp，镜像于 `../mjwarp/index.rst`）—— 独立的 NVIDIA Warp GPU 物理仓库（`notebooks/tutorial.ipynb`、基准和内核源码）。
  - **MuJoCo MPC（MJPC）**（https://github.com/google-deepmind/mujoco_mpc）—— 交互式实时模型预测控制与轨迹优化。
  - **dm_control**（https://github.com/google-deepmind/dm_control）—— Python RL 套件和旧版 `PyMJCF`（见 `../python.rst`，`` `.. _PyMJCF:` ``）。

## 技能目录

| 技能 | 目录 | 主要范围 | 关键 API 与能力 |
| :--- | :--- | :--- | :--- |
| **`mujoco-python`** | [`python/`](python/SKILL.md) | Python 仿真生命周期与物理 | `MjSpec` → `MjModel` → `MjData`、`mj_step`、`mj_forward`、按名访问、`bind()`、`mujoco.batch`（`Batch`）、`mujoco.rollout`、`mujoco.minimize`、`mujoco.sysid`、传感器、`mju_*` 数学。 |
| **`mujoco-spec-editing`** | [`spec_editing/`](spec_editing/SKILL.md) | 程序化模型编写与调优 | 构建 body、geom、joint，带前缀的子 spec 附加，执行器（伺服、电机），碰撞位掩码（`contype`/`conaffinity`），默认类，基线 `<option>`/求解器/接触调优（`.. _CTuning:`）。 |
| **`mujoco-rendering`** | [`rendering/`](rendering/SKILL.md) | 离屏与 PBR 渲染 | 无头 RGB、度量深度图、分割掩码（`mujoco.Renderer`）、Filament PBR（`mujoco.mjrf`、`mujoco.rendering.filament`）、C++ `mjr`/`mjrf` 流水线。 |
| **`mujoco-accelerated`** | [`accelerated/`](accelerated/SKILL.md) | GPU 与 TPU 批量仿真 | 使用 MJX（`MJX-JAX`、`MJX-Warp`、`jax.vmap`、`jax.lax.scan`）的高吞吐批量 rollout，可微性，使用 MJWarp（`mujoco_warp`）的 CUDA 批处理，接触/约束规模设定。 |
| **`mujoco-parity`** | [`parity/`](parity/SKILL.md) | 跨引擎物理一致性 | 比较 CPU（`float64`）与 MJWarp/MJX（`float32`）：逐阶段 `forward` 隔离、刚性执行器与 `qM` 舍入边界、接触匹配、批量单步与 `float32` 基线 rollout。 |
| **`mujoco-gui`** | [`gui/`](gui/SKILL.md) | Dear ImGui 与 ImPlot UI 设计 | 即时模式 UI 模式、布局、样式、ID 栈规则（`##` 与 `###`）、列表虚拟化（`ImGuiListClipper`）、自动化测试钩子。 |
| **`mujoco-studio`** | [`studio/`](studio/SKILL.md) | Studio 应用与交互式查看器 | 独立桌面应用（`.. _StudioApp:`）与 Python 查看器、解耦的 Sim/Viewer 线程、消息通道（快照与事件）、Python 和 C++ 插件架构、热重载、WebViewer WASM 流式传输。 |

## 如何使用这些技能

- **面向 AI 编码代理**：每个技能都包含标准化的 YAML frontmatter，带有 `Use when:` 触发条件和 `Don't use for:` 排除条件，帮助自动化编排器选择相关指南。
- **面向开发者**：技能将快速上手配方和常见陷阱与指向 `doc/*.rst` 参考手册的定向指引配对。