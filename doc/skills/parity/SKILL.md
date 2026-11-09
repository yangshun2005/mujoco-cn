---
name: mujoco-parity
description: >-
  MuJoCo（CPU，float64）与加速后端（MJWarp CUDA、MJX JAX，float32）之间的跨引擎物理一致性对比：逐阶段隔离 mj_forward 与 mjw.forward、归一化容差、刚性仿射执行器与病态质量矩阵（qM）的 float32 边界、执行器条件化前向传播、按 geom 对进行顺序无关的接触匹配、includemargin 接触过滤、批量单步评估，以及 float32 扰动 CPU 轨迹。仅用于对比 MuJoCo CPU 物理与 MuJoCo MJWarp 或 MJX 后端，或调试 MuJoCo 仿真到仿真的轨迹发散、接触缺失或力/qacc 差异。不要用于非 MuJoCo 的 C/C++/CUDA/JAX 物理或 PDE 求解器的 float64 与 float32 一致性测试、通用 GPU 批量设置（mujoco-accelerated）或纯 CPU 仿真（mujoco-python）。
---

# MuJoCo 一致性：对比 CPU（C）与加速（MJWarp / MJX）物理

> [!TIP]
>
> **官方 MuJoCo 文档（`doc/*.rst`）：** 在 MuJoCo 检出目录中时，直接阅读短章节，如
> [`../../mjwarp/index.rst`](../../mjwarp/index.rst)（`.. _mjwDiff:`、
> `.. _mjwOverflow:`、`.. _mjwPairwise:`、`.. _mjwBoxBox:`）、
> [`../../mjx.rst`](../../mjx.rst)（`.. _MjxFeatureParity:`）和
> [`../../programming/simulation.rst`](../../programming/simulation.rst)
> （`.. _siIntegrationState:`、`.. _siWarmstart:`）；对于
> [`../../computation/index.rst`](../../computation/index.rst)（`.. _Pipeline:`、
> `.. _piStages:`）和
> [`../../APIreference/functions.rst`](../../APIreference/functions.rst) 等大文件，grep
> 字面锚点 `.. _Label:`。不在检出目录中时，同一 `doc/*.rst` 树随 `pip install mujoco`
> 离线提供，位于 `pathlib.Path(mujoco.__file__).parent / 'doc'`；否则获取
> `https://mujoco.readthedocs.io/en/latest/<path>.html`（已发布页面用 `/en/stable/`；
> `functions.rst` 渲染在 `APIreference/APIfunctions.html` 中，Sphinx 锚点会将标签转为小写，
> 例如 `.. _mjwDiff:` → `#mjwdiff`）。
>
> **相关技能：**
>
> -   [mujoco-accelerated](../accelerated/SKILL.md) — 使用 MJWarp 和 MJX 进行 GPU/TPU 批量仿真。
> -   [mujoco-python](../python/SKILL.md) — CPU 仿真生命周期、状态访问和步进。

> [!IMPORTANT]
>
> 默认情况下，MuJoCo（C）以 `float64` 执行并动态分配缓冲区，而 MJWarp 和 MJX 以 `float32`
> 执行，预分配接触/约束缓冲区，并使用 GPU 并行碰撞和求解器内核。不考虑 `float32` 条件数、
> 接触顺序和缓冲区容量而直接比较多步原始轨迹，会导致误报和根因误判。若要与 MJWarp/MJX 进行
> 直接的 `float32` 对 `float32` 对比（或通过比较 `float32` CPU 与 `float64` CPU 来隔离纯精度
> 效应），请以单精度（`mjUSESINGLE`）构建 MuJoCo C 及其 Python 绑定：使用
> `--@mujoco//:single_precision`（Bazel）或
> `-DCMAKE_C_FLAGS="-DmjUSESINGLE" -DCMAKE_CXX_FLAGS="-DmjUSESINGLE"`（CMake），
> 并验证 `mujoco.MJTNUM_DTYPE == np.float32`。

## 1. 三级一致性层次

富接触刚体动力学是混沌的：第 $k$ 步一个 $10^{-7}$ 的 `float32` 差异若翻转了单个擦边接触
（`dist ≈ includemargin`）或凸碰撞见证点，此后两条轨迹就会指数发散，即使两个引擎都无 bug。
始终使用三个范围递增的层次来诊断一致性：

| 层次 | 方法 | 隔离内容 | 使用时机 |
| :--- | :--- | :--- | :--- |
| **1. 单状态前向**（`mj_forward` vs `mjw.forward`） | 将相同状态作为 world 批量（`nworld = len(states)`）在两个引擎上评估，按流水线顺序比较中间 `Data` 字段 | 精确定位差异来源的流水线阶段（运动学、肌腱、质量矩阵、执行器、平滑动力学、碰撞或约束求解器）及模型元素 | 主要单元测试和根因诊断工具 |
| **2. 批量单步轨迹**（`mj_step` vs `mjw.step`） | 记录 $T$ 步的参考 CPU 轨迹，然后从所有 $T$ 个步前状态并行执行**一次** MJWarp 步进（`nworld = T`） | 防止误差随时间累积；第 $t$ 步的尖峰直接指向状态 $t$ 的动力学或接触差异，而非更早的漂移 | 沿真实轨迹查找状态相关 bug（如特定接触配置） |
| **3. 带 Float32 基线的开环轨迹** | 从 $t=0$ 起以相同控制步进两个引擎，同时运行 **float32 扰动 CPU 影子轨迹** | 区分真正的一致性 bug（MJWarp 在明确步骤上先于 `float32` CPU 基线发散）与良性混沌发散 | 评估 RL 策略或控制器的端到端轨迹保真度 |

---

## 2. 预检缓冲区大小、溢出检测与状态同步

### 1. 按 CPU 峰值计数的 $2\times$ 设置 `nconmax` 和 `njmax`

当 MJWarp 超过 `nconmax`/`naconmax` 或 `njmax` 时，多余接触或约束行被丢弃，表现与碰撞接触
缺失或求解器失败相同。将容量设为所评估状态中 CPU 最大计数（`d.ncon`、`d.nefc`）的**至少两倍**，
使 MJWarp 产生的额外接触被捕获为接触不匹配，而非被缓冲区溢出截断：

```python
from typing import Optional, Sequence
import mujoco
import mujoco_warp as mjw
import numpy as np
import warp as wp

MIN_NCONMAX = 64
MIN_NJMAX = 256


def capacities(
    mjds: Sequence[mujoco.MjData],
    nconmax: Optional[int] = None,
    njmax: Optional[int] = None,
) -> tuple[int, int]:
  if nconmax is None:
    nconmax = max(MIN_NCONMAX, 2 * max(d.ncon for d in mjds))
  if njmax is None:
    njmax = max(MIN_NJMAX, 2 * max(d.nefc for d in mjds))
  return nconmax, njmax


def check_overflow(d: mjw.Data) -> None:
  nacon = int(d.nacon.numpy()[0])
  if nacon > d.naconmax:
    raise RuntimeError(
        f"MJWarp contact overflow ({nacon} > naconmax {d.naconmax}); "
        "increase nconmax."
    )
  nefc = int(d.nefc.numpy().max())
  if nefc > d.njmax:
    raise RuntimeError(
        f"MJWarp constraint overflow ({nefc} > njmax {d.njmax}); "
        "increase njmax."
    )
  buf_mask = ~(mjw.OverflowType.ITERATIONS | mjw.OverflowType.LS_ITERATIONS)
  overflow = int(np.bitwise_or.reduce(d.overflow.numpy()) & buf_mask)
  if overflow:
    raise RuntimeError(
        f"MJWarp buffer overflow: {mjw.OverflowType(overflow)!r}"
    )
```

### 2. 同步完整仿真状态（`set_states`）

仅复制 `qpos` 和 `qvel` 会使执行器激活、mocap 位姿、施加的力旋量、等式约束开关、延迟历史缓冲区和
求解器热启动不同步。特别是，除非设置了 `mjDSBL_WARMSTART`，MJWarp 直接从 `qacc_warmstart`
初始化求解器（而 CPU MuJoCo 将 `qacc_warmstart` 与 `qacc_smooth` 比较）。使用
`mujoco.mj_getState` 和 `mjw.set_state` 配合 `mjSTATE_INTEGRATION` 将完整积分状态复制到各 world：

```python
def copy_data(mjm: mujoco.MjModel, src: mujoco.MjData) -> mujoco.MjData:
  dst = mujoco.MjData(mjm)
  mujoco.mj_copyData(dst, mjm, src)
  return dst


def set_states(
    mjm: mujoco.MjModel,
    m: mjw.Model,
    d: mjw.Data,
    states: Sequence[mujoco.MjData],
) -> None:
  """Copies the full integration state of each CPU MjData into the corresponding MJWarp world."""
  if len(states) != d.nworld:
    raise ValueError(f"Got {len(states)} states for {d.nworld} worlds.")
  sig = int(mujoco.mjtState.mjSTATE_INTEGRATION)
  size = mujoco.mj_stateSize(mjm, sig)
  mj_states = np.empty((len(states), size))
  for i, s in enumerate(states):
    mujoco.mj_getState(mjm, s, mj_states[i], sig)
  mjw.set_state(m, d, wp.array(mj_states, dtype=float), sig)


def put_batched(
    mjm: mujoco.MjModel,
    mjds: Sequence[mujoco.MjData],
    nconmax: Optional[int] = None,
    njmax: Optional[int] = None,
) -> tuple[mjw.Model, mjw.Data]:
  nconmax, njmax = capacities(mjds, nconmax, njmax)
  m = mjw.put_model(mjm)
  d = mjw.put_data(mjm, mjds[0], nworld=len(mjds), nconmax=nconmax, njmax=njmax)
  set_states(mjm, m, d, mjds)
  return m, d
```

---

## 3. 逐阶段 `forward` 隔离与条件数边界

### 1. 流水线阶段顺序与基线容差

`mj_forward` 的每个阶段为下一阶段提供输入。始终按拓扑流水线顺序检查误差——归一化误差 $> 1$
的**第一个**阶段即为根因：

| 流水线阶段 | `Data` 字段 | 默认 `atol` | 默认 `rtol` | 备注 |
| :--- | :--- | :--- | :--- | :--- |
| **1. 运动学** | `xpos`、`xquat`、`site_xpos`、`subtree_com` | `1e-5` | `1e-4` | 正向运动学（`mj_kinematics`、`mj_comPos`） |
| **2. 肌腱与长度** | `ten_length`、`actuator_length` | `1e-5` | `1e-4` | 空间/固定肌腱与传动长度 |
| **3. 质量矩阵** | `M` | `1e-8` | `1e-5` | CSR 下三角（`mjm.M_rownnz`、`mjm.M_colind`）；添加 CRB 尺度边界 |
| **4. 速度与力** | `actuator_force`、`qfrc_actuator`、`qfrc_bias`、`qfrc_passive` | `1e-3` | `1e-3` | RNE 偏置、被动弹簧/阻尼器、仿射执行器；添加刚性执行器边界 |
| **5. 平滑动力学** | `qfrc_smooth`、`qacc_smooth` | `1e-3`、`1e-2` | `1e-3` | 无约束净力和 $M^{-1} q_{\text{frc,smooth}}$；在执行器条件化 CPU `MjData` 上比较 |
| **6. 约束动力学** | `qfrc_constraint`、`qacc` | `1e-1` | `1e-2` | 迭代求解器输出（CG/Newton 在 `opt.tolerance` 停止）；在执行器条件化 CPU `MjData` 上比较 |

归一化误差定义为：

```text
err = |mujoco - warp| / (atol + rtol * |mujoco| + extra_atol)
```

值 $\le 1.0$ 即在容差内。

### 2. 陷阱 1：刚性仿射执行器与执行器条件化

高增益位置或肌腱伺服（如 $k_p \sim 10^4\text{--}10^6$）将力计算为大项之差：
$f = (g_0 + g_1 L + g_2 \dot{L}) u + (b_0 + b_1 L + b_2 \dot{L})$。$L$（`actuator_length`）
或 $\dot{L}$（`actuator_velocity`）的 `float32` 舍入乘以 $g$ 和 $b$，产生大小为
$O(\epsilon_{32} (|g| |u| + |b|))$ 的合法力差异，超过 `1e-3`。当这些力作用于轻自由度（如手指）时，
会在 `qacc_smooth`、`qfrc_constraint` 和 `qacc` 中造成大的下游差异，伪装成质量矩阵或约束求解器 bug。

**两部分修复：**
1. 为 `actuator_force` 和 `qfrc_actuator` 添加逐分量 `float32` 舍入边界（`actuator_rounding_atol`）。
2. 检查下游字段（`qfrc_smooth`、`qacc_smooth`、`qfrc_constraint`、`qacc`）时，**用 MJWarp 的执行器力条件化 CPU 参考**：
   将 `warp_qfrc_actuator - mjd.qfrc_actuator` 加入 `mjd.qfrc_applied` 并重新运行 `mj_forward`：

```python
FLOAT32_ROUNDING_FACTOR = 8.0
ACTUATION_DOWNSTREAM_FIELDS = frozenset(
    {"qfrc_smooth", "qacc_smooth", "qfrc_constraint", "qacc"}
)


def actuator_rounding_atol(
    mjm: mujoco.MjModel, mjd: mujoco.MjData
) -> dict[str, np.ndarray]:
  """Returns float32 rounding bounds for actuator_force and qfrc_actuator."""
  if not mjm.nu or (mjm.opt.disableflags & mujoco.mjtDisableBit.mjDSBL_ACTUATION):
    return {}
  eps = float(np.finfo(np.float32).eps)
  length, velocity = mjd.actuator_length, mjd.actuator_velocity
  gain, bias = mjm.actuator_gainprm, mjm.actuator_biasprm
  gaintype, biastype = mjm.actuator_gaintype, mjm.actuator_biastype

  ctrl = mjd.ctrl.copy()
  if not (mjm.opt.disableflags & mujoco.mjtDisableBit.mjDSBL_CLAMPCTRL):
    limited = mjm.actuator_ctrllimited.astype(bool)
    ctrl[limited] = np.clip(
        ctrl[limited],
        mjm.actuator_ctrlrange[limited, 0],
        mjm.actuator_ctrlrange[limited, 1],
    )
  act_input = np.abs(ctrl)
  stateful = mjm.actuator_actnum > 0
  if np.any(stateful):
    act_ind = mjm.actuator_actadr[stateful] + mjm.actuator_actnum[stateful] - 1
    act = mjd.act[act_ind].copy()
    act_lim = mjm.actuator_actlimited[stateful].astype(bool)
    act[act_lim] = np.clip(
        act[act_lim],
        mjm.actuator_actrange[stateful][act_lim, 0],
        mjm.actuator_actrange[stateful][act_lim, 1],
    )
    act_input[stateful] = np.abs(act)

  gain_terms = np.abs(gain[:, 0]) + np.where(
      gaintype == mujoco.mjtGain.mjGAIN_AFFINE,
      np.abs(gain[:, 1] * length) + np.abs(gain[:, 2] * velocity),
      0.0,
  )
  bias_terms = np.where(
      biastype == mujoco.mjtBias.mjBIAS_AFFINE,
      np.abs(bias[:, 0])
      + np.abs(bias[:, 1] * length)
      + np.abs(bias[:, 2] * velocity),
      0.0,
  )
  affine = np.isin(
      gaintype,
      [int(mujoco.mjtGain.mjGAIN_FIXED), int(mujoco.mjtGain.mjGAIN_AFFINE)],
  ) & np
```