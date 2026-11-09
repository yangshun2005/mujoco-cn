---
name: mujoco-accelerated
description: >-
  使用 MJX（JAX 中的 mujoco.mjx）和 MJWarp（CUDA 中的 mujoco_warp）进行 MuJoCo GPU 与 TPU 批量仿真：并行 rollout
  （jax.vmap）、轨迹 scan（jax.lax.scan）、可微性、设备内存
  （put_model、put_data）、静态接触尺寸设定（max_contact_points、
  max_geom_pairs、nconmax、njmax）、float32 精度、XLA 编译缓存。
  仅用于 MuJoCo MJX 或 MJWarp 的 GPU/TPU 批量 rollout、RL 环境以及
  可微 MuJoCo 物理仿真。不要用于自定义的非 MuJoCo JAX 或 CUDA
  Warp 仿真（例如自定义 2D 游戏、分子动力学、轨道力学、
  DEM 颗粒求解器，或未导入 `mujoco.mjx` 或
  `mujoco_warp` 的浅水 PDE）、CPU 仿真（mujoco-python）、跨引擎一致性调试
  （mujoco-parity）、渲染（mujoco-rendering）或 GUI（mujoco-gui）。
---

# MuJoCo 加速：MJX 与 MJWarp GPU 后端

> [!TIP]
>
> **官方 MuJoCo 文档（`doc/*.rst`）：** 在 MuJoCo
> 检出目录中时，直接阅读
> [`../../mjx.rst`](../../mjx.rst)（`.. _Mjx:`、`.. _MjxWarp:`、`.. _MjxJAX:`）、
> [`../../mjx_api.rst`](../../mjx_api.rst)、
> [`../../mjwarp/index.rst`](../../mjwarp/index.rst)（`.. _MJW:`）以及
> [`../../mjwarp/api.rst`](../../mjwarp/api.rst)
> （对 `python.rst` 这类大文件，保留用 `.. _Label:` grep 的方式）。
> 在检出目录之外，同一套 `doc/*.rst` 目录树会离线随
> `pip install mujoco` 发布，位于 `pathlib.Path(mujoco.__file__).parent / 'doc'`；
> 否则请访问 `https://mujoco.readthedocs.io/en/latest/<path>.html`（已发布页面用
> `/en/stable/`；Sphinx 锚点会将标签转为小写，例如
> `.. _PyBatch:` → `#pybatch`），并在
> `https://github.com/google-deepmind/mujoco/tree/main/` 下浏览仓库路径（`../../../`）。
>
> **相关技能：**
>
> -   [mujoco-python](../python/SKILL.md) — CPU 物理仿真、运动学
>     以及完整求解器套件。
> -   [mujoco-parity](../parity/SKILL.md) — CPU MuJoCo 与 MJWarp/MJX 之间的
>     跨引擎物理一致性检查（`forward` 隔离、接触匹配
>     以及 `float32` 基线 rollout）。
> -   [mujoco-studio](../studio/SKILL.md) — 交互式仿真与 Studio
>     查看器插件。

> [!IMPORTANT]
>
> MJX 和 MJWarp 是**仅限 Python** 的加速后端。标准 CPU
> 仿真通过 `import mujoco` 以完整的 `float64` 运行。

## 1. 后端对比：C++ vs MJX-JAX vs MJX-Warp vs MJWarp

| 特性 | C++ 引擎（默认） | MJX-JAX（`impl='jax'`） | MJX-Warp（`impl='warp'`） | MJWarp（CUDA） |
| :--- | :--- | :--- | :--- | :--- |
| **导入** | `import mujoco`（`mujoco.batch`、`mujoco.rollout`） | `from mujoco import mjx` | `from mujoco import mjx` | `import mujoco_warp as mjw` |
| **硬件** | CPU | CPU / GPU / TPU | NVIDIA GPU | NVIDIA GPU |
| **精度** | float64 | float32 | float32 | float32 |
| **可微** | ❌ 否 | ✅ 是（JAX 梯度） | ❌ 否 | ❌ 否 |
| **批量步进** | ✅ `mujoco.batch.Batch` 与 `mujoco.rollout`（CPU 线程） | ✅ `jax.vmap` | ✅ `jax.vmap`（CUDA 内核） | ✅ 原生 CUDA 批量 |
| **求解器** | Newton、CG、PGS、**noslip** | Newton、CG | Newton、CG | Newton、CG |
| **约束岛**| ✅ 是 | ❌ 否 | ✅ 是 | ✅ 是 |
| **插件** | 所有引擎插件 | ❌ 否 | 仅 SDF 插件 | 仅 SDF 插件 |

完整特性一致性以及求解器/碰撞细节见 [mjx.rst](../../mjx.rst)（`.. _MjxWarp:`、`.. _MjxJAX:`、
`.. _MjxFeatureParity:`）和 [mjwarp/index.rst](../../mjwarp/index.rst)
（`.. _mjwDiff:`）。如需支持 `float64`、`noslip`/`PGS` 以及
所有引擎插件、且不依赖 JAX 或 CUDA 的多线程 CPU 批量仿真，请使用有状态的 `mujoco.batch.Batch`（用于
交互式 RL 步进、子集重置以及逐仿真域随机化——见
[python.rst](../../python.rst)，`.. _PyBatch:`）或无状态的 `mujoco.rollout`
（`.. _PyRollout:`，以及 [programming/simulation.rst](../../programming/simulation.rst)，
`.. _siBatch:`）。

## 2. MJX 工作流（JAX）

MJX 将 MuJoCo 数据结构转换为不可变的 JAX pytree，可在 GPU 和 TPU 上原生执行。

```python
import mujoco
from mujoco import mjx

# 1. Load CPU model and data
model = mujoco.MjModel.from_xml_path('scene.xml')
data = mujoco.MjData(model)

# 2. Transfer to device (GPU/TPU)
mjx_model = mjx.put_model(model)
mjx_data = mjx.put_data(model, data)

# 3. Step on device (functional paradigm returns new data)
mjx_data = mjx.step(mjx_model, mjx_data)

# 4. Copy data back to host CPU
mjx.get_data_into(data, model, mjx_data)
print(data.body('torso').xpos)
```

在 NVIDIA GPU 上，若不需要 JAX 自动微分，可向
`mjx.put_model(model, impl='warp')` 和
`mjx.make_data(model, impl='warp', naconmax=..., njmax=...)` 传入 `impl='warp'`（`pip install
mujoco-mjx[warp]`），以在 JAX 内运行 MJWarp 的 CUDA 接触/约束内核
（`jax.jit` / `jax.vmap`）；使用 `Data.where(...)` 进行逐世界条件重置，
并阅读 [mjx.rst](../../mjx.rst)（`.. _MjxWarp:`、`.. _MjxWarpGraphModes:`、
`.. _MjxPerformanceWarp:`）了解 CUDA 图模式和批量渲染。

## 3. 高吞吐 Rollout：jax.vmap 与 jax.lax.scan

### 并行批量（`jax.vmap`）

```python
import jax
from mujoco import mjx

# Vectorize step over batch dimension (model unbatched, data batched)
batched_step = jax.vmap(mjx.step, in_axes=(None, 0))

# JIT-compile for maximum device throughput
batched_step_jit = jax.jit(batched_step)
batch_data = batched_step_jit(mjx_model, batch_data)
```

### 多步轨迹 Rollout（`jax.lax.scan`）

对于 RL 环境步进，避免使用 Python 循环，改为跨步 scan：

```python
def rollout(model: mjx.Model, init_data: mjx.Data, ctrl_sequence: jax.Array):
  def step_fn(d, ctrl):
    d = d.replace(ctrl=ctrl)
    d = mjx.step(model, d)
    return d, (d.qpos, d.sensordata)

  final_data, trajectory = jax.lax.scan(step_fn, init_data, ctrl_sequence)
  return final_data, trajectory
```

## 4. 静态接触与约束尺寸设定

JAX 要求静态数组形状，因此接触和约束缓冲区在创建设备数据时确定大小，而非每步确定。

### MJX（默认 JAX 实现）

接触缓冲区的大小**由模型决定**，而非由 `data.ncon` 决定：每个可能碰撞的几何体对
（经 `contype`/`conaffinity`、`exclude` 和 `pair`
过滤后）都会获得固定数量的接触槽位，因此大型场景会产生庞大的 `efc_*` 数组并导致编译缓慢。若 CPU `MjData` 已持有的接触数超过此容量，
`put_data` 会抛出 `ValueError`。
此处 `mjx.put_data` 的 `njmax` 参数会被忽略。

要缩小缓冲区，可在 MJCF 中用自定义数值设定上限：

```xml
<custom>
  <!-- keep only the N closest geom pairs (bounding spheres) per geom-type pair -->
  <numeric name="max_geom_pairs" data="16"/>
  <!-- keep only the N deepest contacts per condim group -->
  <numeric name="max_contact_points" data="32"/>
</custom>
```

同时通过 `contype`/`conaffinity` 和
`<exclude>` 减少候选对数量，并优先使用基本几何体而非网格。

### MJWarp（以及 `mjx.put_data(..., impl='warp')`）

缓冲区由显式参数确定大小；省略的值回退到由模型推导的默认值：

-   `nconmax`：每个世界的接触数（MJWarp `put_data`/`make_data`）。
-   `naconmax`：所有世界的接触总数。
-   `njmax`：**每个世界**的约束行数（`efc`），即每个世界的硬上限。

若输入 `MjData` 超出容量，`put_data` 会抛出异常。步进过程中，
多余的接触/约束会被丢弃，打印警告，并设置逐世界的
`d.overflow` 位掩码（`mjw.OverflowType`），因此请按峰值设定大小。

## 5. MJX 命名访问与 `bind()`

MJX 通过 `bind()` 提供对 JAX 数组的声明式命名访问：

```python
spec = mujoco.MjSpec.from_file('model.xml')
model = spec.compile()
data = mujoco.MjData(model)

mx = mjx.put_model(model)
dx = mjx.put_data(model, data)

# Read body positions from JAX device arrays
torso_pos = dx.bind(mx, spec.body('torso')).xpos

# Functional update of control inputs
dx = dx.bind(mx, spec.actuators).set('ctrl', [1.0, 0.5])
```

## 6. MJWarp 指南（NVIDIA CUDA）

MJWarp（`mujoco_warp`）提供 CUDA 原生仿真器，针对 NVIDIA 硬件上的最大批量吞吐量优化，不依赖 JAX：

```python
import mujoco
import mujoco_warp as mjw

model = mujoco.MjModel.from_xml_path('scene.xml')
data = mujoco.MjData(model)

# Transfer to CUDA: one model, nworld copies of the data
mjw_model = mjw.put_model(model)
mjw_data = mjw.put_data(model, data, nworld=4096, nconmax=64, njmax=256)
# or allocate fresh: mjw.make_data(model, nworld=4096, nconmax=64, njmax=256)

# Advance all 4096 environments in parallel (in place)
mjw.step(mjw_model, mjw_data)

# Fields are Warp arrays with a leading world dimension
qpos = mjw_data.qpos.numpy()  # (4096, nq)

# Copy one world back to CPU MjData
mjw.get_data_into(data, model, mjw_data, world_id=0)
```

## 7. 特性一致性、调优与注意事项

选择碰撞器、求解器或调优吞吐量时，不要依赖静态特性快照，而应查阅 RST 手册中权威的一致性
与性能章节：

-   **支持的几何体、求解器及与 CPU MuJoCo 的差异**：
    [mjx.rst](../../mjx.rst)（`.. _MjxFeatureParity:`、`.. _MjxSharpBits:`）和
    [mjwarp/index.rst](../../mjwarp/index.rst)（`.. _mjwDiff:`、
    `.. _mjwPairwise:`、`.. _mjwBoxBox:`）。值得注意的是，`noslip` 和 `PGS` 求解器
    仅限 CPU，且 MJX-JAX 以整体方式求解约束，不使用约束岛。
-   **性能调优、CUDA 图与域随机化**：
    [mjx.rst](../../mjx.rst)（`.. _MjxPerformance:`、`.. _MjxPerformanceJAX:`、
    `.. _MjxPerformanceWarp:`）、[mjwarp/index.rst](../../mjwarp/index.rst)
    （`.. _mjwPerf:`、`.. _mjwGC:`、`.. _mjwOverflow:`、`.. _mjwBatch:`）以及
    [modeling.rst](../../modeling.rst)（`.. _CTuning:`、`.. _CTuningGlobal:`、
    `.. _CTuningModel:`）了解通用时间步、求解器/迭代、摩擦锥
    以及基本几何体与网格碰撞的权衡。

### 高频注意事项

1.  **Float32 精度**：GPU 后端以单精度（`float32`）运行。
    刚性系统可能出现数值漂移或 NaN 错误。可通过
    增大 `model.opt.iterations` 或减小 `model.opt.timestep` 来抵消。
    若要将 `float32` GPU 物理与 CPU MuJoCo 对比，请参阅
    [`mujoco-parity`](../parity/SKILL.md)。
2.  **MJX 中的不可变性**：`mjx.Data` 是 JAX pytree。原地赋值
    会失败；请使用 `.replace()`：
    ```python
    mjx_data = mjx_data.replace(qpos=new_qpos)
    ```
3.  **编译缓存**：启用持久化 XLA 缓存，避免启动时长时间重新编译：
    ```python
    import jax
    jax.config.update('jax_compilation_cache_dir', '/tmp/jax_cache')
    ```

## 8. 关键参考

### 文档

-   [MJX 指南（`MJX-JAX` 与 `MJX-Warp`）](../../mjx.rst)（`.. _Mjx:`、
    `.. _MjxWarp:`、`.. _MjxJAX:`、`.. _MjxFeatureParity:`、
    `.. _MjxPerformance:`、`.. _MjxSharpBits:`）以及
    [MJX API 参考](../../mjx_api.rst)
-   [MuJoCo Warp（MJWarp）指南](../../mjwarp/index.rst)（`.. _MJW:`、
    `.. _MJW_Usage:`、`.. _mjwPerf:`、`.. _mjwOverflow:`、`.. _mjwBatch:`、
    `.. _mjwDiff:`）以及 [MJWarp API 参考](../../mjwarp/api.rst)
-   [Python 绑定（`mujoco.batch` 与 `mujoco.rollout`）](../../python.rst)
    （`.. _PyNamed:`、`.. _PyBatch:`、`.. _PyRollout:`）、
    [模型调优指南](../../modeling.rst)（`.. _CTuning:`）以及
    [MuJoCo Menagerie](../../models.rst)（`.. _Menagerie:`）

### Notebook、模型与 RL 生态

-   **教程与测试**：`mjx/tutorial.ipynb`（[mjx.rst](../../mjx.rst) 中的
    `.. _MjxNotebook:`）、
    [MJX 支持测试](../../../mjx/mujoco/mjx/_src/support_test.py)、
    [Humanoid 模型](../../../model/humanoid/humanoid.xml)以及
    [基准测试模型](../../../test/benchmark/testdata/100_humanoids_chol.xml)
-   **RL 与仿真生态**（见
    [mjwarp/index.rst](../../mjwarp/index.rst)，`.. _mjwFAQ:`）：
    -   [MuJoCo Playground（`google-deepmind/mujoco_playground`）](https://github.com/google-deepmind/mujoco_playground)
        — 用于运动、操作、灵巧手和视觉的 JAX/MJX（`MJX-JAX` 与 `MJX-Warp`）RL 环境。
    -   [`mjlab`（`mujocolab/mjlab`）](https://github.com/mujocolab/mjlab) —
        Isaac Lab 风格的 manager API，可直接用
        `mujoco_warp` 和 PyTorch 训练策略。
    -   [MuJoCo Warp（`google-deepmind/mujoco_warp`）](https://github.com/google-deepmind/mujoco_warp)
        — 独立的 `mujoco_warp` 仓库（`notebooks/tutorial.ipynb`、
        基准测试和 Warp