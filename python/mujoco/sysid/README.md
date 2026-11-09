# 系统辨识工具箱

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.png)](https://colab.research.google.com/github/google-deepmind/mujoco/blob/main/python/mujoco/sysid/sysid.ipynb)

给定 MuJoCo 模型和录制的传感器数据，找出使仿真与现实匹配的参数。默认情况下，该库使用带箱约束的非线性最小二乘法，最小化测量输出与仿真（预测）输出之间的差异。残差可由静态参数或优化参数（如权重和时间延迟）修改。

优化器使用 Gauss-Newton 方法和有限差分 Jacobian。每次参数扰动都需要一次独立的仿真 rollout。所有 rollout 在单次批量调用 `mujoco.rollout` 中执行，跨线程并行化。

## 流水线

**你需要提供：**
- 一个或多个 `ModelSequences`，每个捆绑一个 `MjSpec` 和一组或多组测量数据序列。所有内容将联合优化。
- 一个 `ParameterDict`，定义带边界可微的 `Parameter`。
- 将 `Parameter` 应用到 `MjSpec` 的回调（单独或联合应用）。
- （可选）覆盖默认残差函数行为的函数（`build_model`、`custom_rollout`、`modify_residual`）。

**框架负责：**
- 将用户代码组合为残差函数（`build_residual_fn`）。
- 通过批量并行 rollout 优化参数（`optimize`）。
- 保存结果并生成 HTML 报告（`save_results`、`default_report`）。

## 可以辨识什么？

你可以优化任何能以可微方式修改最终残差的参数。常见用例包括：

**物理参数**，可在 `MjSpec` 上设置。MjSpec 中的大多数参数可通过用户提供的回调直接设置：

| 目标 | 方法 |
|---|---|
| 接触滑动摩擦 | `spec.pair("cp").friction[0] = p.value[0]` |
| 关节阻尼 | `spec.joint("j1").damping = p.value[0]` |

对于无法简单应用到 MjSpec 的常见系统辨识参数化，提供了便捷函数：

| 目标 | 方法 |
|---|---|
| 刚体质量 | `body_inertia_param(..., InertiaType.Mass)` |
| 刚体质量 + 质心 | `body_inertia_param(..., InertiaType.MassIpos)` |
| 完整惯性（10 维） | `body_inertia_param(..., InertiaType.Pseudo)` |
| 执行器 P/D 增益 | `apply_pdgain(spec, "act1", p.value)` |

完整惯性使用伪惯性 Cholesky 参数化（[Rucker & Wensing 2022](https://ieeexplore.ieee.org/document/9690029)），保证物理一致性且无奇异性。

**测量参数**（如传感器延迟、增益和偏置）是测量系统的属性，不是物理模型的属性。该库提供了在 rollout 后将此类修正应用到残差的工具。这些工具功能完整，但 API 尚未最终确定。

## 示例

```python
import mujoco
from mujoco import sysid

# 1. 加载模型并定义参数。
spec = mujoco.MjSpec.from_file("robot.xml")
model = spec.compile()

def set_link1_mass(spec, p):
    spec.body("link1").mass = p.value[0]

params = sysid.ParameterDict()
params.add(sysid.Parameter(
    "link1_mass", nominal=2.0, min_value=0.5, max_value=5.0,
    modifier=set_link1_mass))

# 2. 加载并打包测量数据。
# 假设数组按 MuJoCo 顺序排列，否则传入 names 参数
control = sysid.TimeSeries.from_control_names(times, ctrl_array, model)
measureddata = sysid.TimeSeries.from_names(times, measurement_array, model)
initial_state = sysid.create_initial_state(model, qpos_0, qvel_0)
ms = sysid.ModelSequences("robot", spec, "traj_1", initial_state, control, measureddata)

# 3. 构建残差、优化、保存。
residual_fn = sysid.build_residual_fn(models_sequences=[ms])
opt_params, opt_result = sysid.optimize(initial_params=params, residual_fn=residual_fn)
sysid.save_results("results/", [ms], params, opt_params, opt_result, residual_fn)
```

`default_report` 生成交互式 HTML 报告，包含视频、测量对比、参数表和置信区间。