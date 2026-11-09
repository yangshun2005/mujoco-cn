---
name: mujoco-python
description: >-
  MuJoCo Python（`import mujoco`）仿真生命周期（MjSpec、MjModel、MjData）、
  步进（mj_step、mj_step1、mj_step2、mj_forward）、命名访问（data.body、
  model.geom）、bind()、多线程 CPU 批量仿真（mujoco.batch.Batch、
  mujoco.rollout）、优化与系统辨识（mujoco.minimize、mujoco.sysid、
  mjd_transitionFD）、接触/触摸传感器、sensordata、关键帧重置、
  积分器、mju_ 空间数学（四元数、位姿）以及能量诊断。
  仅用于 MuJoCo Python 物理仿真、MjModel/MjData 状态访问、
  MuJoCo CPU 批处理、步进循环以及 MuJoCo 传感器/系统辨识。不要用于
  自定义的非 MuJoCo Python/NumPy/C++ ODE 或刚体仿真器、自定义
  有限差分系统辨识，或未使用 `mujoco` 的通用空间数学，也不要用于
  规格编辑（mujoco-spec-editing）、渲染（mujoco-rendering）、GPU/TPU
  （mujoco-accelerated）、一致性对比（mujoco-parity）或 GUI/查看器（mujoco-gui、
  mujoco-studio）。
---

# MuJoCo Python：仿真、传感器与数学

> [!TIP]
>
> **MuJoCo 官方文档（`doc/*.rst`）：** 在 MuJoCo 检出目录中时，直接阅读
> [`../../programming/simulation.rst`](../../programming/simulation.rst)
> （`.. _Simulation:`）和 [`../../overview.rst`](../../overview.rst) 等短章节；
> 对于大型参考文件，如
> [`../../python.rst`](../../python.rst)（`.. _PyStructs:`、`.. _PyNamed:`、
> `.. _PyBatch:`、`.. _PySysid:`）、
> [`../../modeling.rst`](../../modeling.rst)（`.. _CTuning:`）、
> [`../../computation/index.rst`](../../computation/index.rst)（`.. _Pipeline:`、
> `.. _geIntegrators:`、`.. _geChoosingTimestepIntegrator:`）、
> [`../../XMLreference.rst`](../../XMLreference.rst) 和
> [`../../APIreference/functions.rst`](../../APIreference/functions.rst)，请 grep
> 字面量 `.. _Label:` 锚点。在检出目录之外，同一 `doc/*.rst` 目录树会离线随
> `pip install mujoco` 发布，位于
> `pathlib.Path(mujoco.__file__).parent / 'doc'`；否则请获取
> `https://mujoco.readthedocs.io/en/latest/<path>.html`（已发布页面用 `/en/stable/`；
> `functions.rst` 渲染在
> `APIreference/APIfunctions.html` 中，Sphinx 锚点会将标签转为小写，
> 例如 `.. _PyBatch:` → `#pybatch`），并在
> `https://github.com/google-deepmind/mujoco/tree/main/` 下浏览仓库路径（`../../../`、
> `python/*.ipynb`）。
>
> **相关技能：**
>
> -   [mujoco-spec-editing](../spec_editing/SKILL.md) — 使用 `MjSpec` 进行程序化模型
>     编辑和场景组合。
> -   [mujoco-rendering](../rendering/SKILL.md) — 离屏图像渲染、
>     相机、深度和分割。
> -   [mujoco-accelerated](../accelerated/SKILL.md) — 使用 MJX（JAX）和 MJWarp（CUDA）进行
>     GPU/TPU 物理仿真。
> -   [mujoco-parity](../parity/SKILL.md) — 与 MJWarp 和 MJX 进行
>     跨引擎物理一致性对比。
> -   [mujoco-studio](../studio/SKILL.md) — 交互式查看器、Studio 查看器
>     应用和插件架构。

> [!CAUTION]
>
> -   **绝不要使用 `mj_name2id`、`model.name2id` 或原始整数索引**
>     （例如 `data.qpos[0]`、`data.xpos[body_id]`）按名称查找元素。
>     始终改用**命名访问**（`data.body('name').xpos`）或 **`bind()`**。
> -   **不要使用 dm_control / PyMJCF 的查找模式。** `model.find('body',
>     'name')`、`.find_body()` 和 `physics.named` 在
>     MuJoCo 3.x 中**不存在**。请使用直接访问器：在
>     `MjSpec`、`MjModel` 和 `MjData` 上调用 `.body('name')`、`.geom('name')` 等。

## 1. 编译生命周期：MjSpec → MjModel → MjData

MuJoCo 模型遵循三阶段生命周期：

```text
MjSpec  ──spec.compile()──▶  MjModel  ──MjData(model)──▶  MjData
  │                             │                            │
  │  (mutable blueprint)        │  (compiled, mostly frozen) │ (simulation state)
  │                             │                            │
  └── spec.recompile(m, d) ─────┴────────────────────────────┘
```

1.  **`MjSpec`**：定义仿真元素和层级的可变数据结构。
2.  **`MjModel`**：编译、优化后的物理模型，包含常量和预计算参数。结构变更需要重新编译。
3.  **`MjData`**：动态仿真状态（位置、速度、力、接触、诊断缓冲区）。

### Python 生命周期

```python
import mujoco

# 1. Build or load spec
spec = mujoco.MjSpec()
body = spec.worldbody.add_body(name='pendulum', pos=[0, 0, 1])
body.add_geom(type=mujoco.mjtGeom.mjGEOM_SPHERE, size=[0.1])
body.add_joint(name='hinge', type=mujoco.mjtJoint.mjJNT_HINGE, axis=[0, 1, 0])

# 2. Compile to model and instantiate data
model = spec.compile()
data = mujoco.MjData(model)
mujoco.mj_forward(model, data)

# 3. Dynamic recompilation (preserves existing state)
body2 = spec.worldbody.add_body(name='ball2', pos=[1, 0, 1])
body2.add_geom(size=[0.1])
body2.add_freejoint()
model, data = spec.recompile(model, data)  # Always reassign return values
```

### C++ 生命周期

```cpp
#include <mujoco/mujoco.h>

// 1. Create or load spec (or: mj_parseXML(filename, nullptr, error, sizeof(error)))
mjSpec* spec = mj_makeSpec();
mjsBody* world = mjs_findBody(spec, "world");
mjsBody* body = mjs_addBody(world, nullptr);
mjsGeom* geom = mjs_addGeom(body, nullptr);
geom->type = mjGEOM_SPHERE;
geom->size[0] = 0.1;

// 2. Compile to model and create data
mjModel* model = mj_compile(spec, nullptr);
if (!model) {
  printf("%s\n", mjs_getError(spec));
}
mjData* data = mj_makeData(model);

// 3. Step or evaluate
mj_forward(model, data);

// 4. Clean up allocated resources
mj_deleteData(data);
mj_deleteModel(model);
mj_deleteSpec(spec);
```

### 加载与序列化

```python
# Direct model loading from file or string
model = mujoco.MjModel.from_xml_path('model.xml')
model = mujoco.MjModel.from_xml_string(xml_str)
model = mujoco.MjModel.from_binary_path('model.mjb')  # Precompiled binary format

# Loading mutable spec from file or string
spec = mujoco.MjSpec.from_file('model.xml')
spec = mujoco.MjSpec.from_string(xml_str)

# Serialization
xml_string = spec.to_xml()
spec.encode('model.xml', model)
```

> [!CAUTION]
>
> `mujoco.load_model_from_path` 在 MuJoCo 3.x 中**不存在**。请使用
> `mujoco.MjModel.from_xml_path(path)` 或 `mujoco.MjSpec.from_file(path)`。
> 可直接运行的模型见 `../../../model/`（`humanoid/humanoid.xml`、
> `tendon_arm/arm26.xml`、`car/car.xml`）、`../../../test/engine/testdata/` 以及
> MuJoCo Menagerie（[models.rst](../../models.rst)，`.. _Menagerie:`）。

## 2. 访问状态：命名访问与 bind()

### 1. 命名访问（在 `MjModel` 和 `MjData` 上）

在 Python 中，通过调用元素类型访问器并传入元素名称来访问元素：

```python
# Reading state
torso_pos = data.body('torso').xpos      # NumPy view (3,)
knee_angle = data.joint('knee').qpos     # Shape depends on joint type
ball_size = model.geom('ball').size      # NumPy view (3,)

# Writing controls and state
data.actuator('motor').ctrl = 1.0        # Writable view
data.joint('knee').qpos = 0.5
```

支持的元素访问器：`body`、`geom`、`joint`（别名 `jnt`）、`site`、
`camera`（别名 `cam`）、`light`、`mesh`、`skin`、`texture`（别名 `tex`）、
`material`（别名 `mat`）、`pair`、`equality`（别名 `eq`）、`tendon`（别名
`ten`）、`actuator`、`sensor`、`numeric`、`text`、`tuple`、`keyframe`（别名
`key`）。

> [!WARNING]
>
> **命名访问返回视图，而非副本。** 在仿真循环中，修改仿真会原地改变数组。
> 如果记录或存储历史，**务必调用 `.copy()`**：
>
> ```python
> # ❌ WRONG — all history items will point to the same final state
> history.append(data.body('torso').xpos)
>
> # ✅ RIGHT — records distinct coordinates
> history.append(data.body('torso').xpos.copy())
> ```

### 2. `bind()`（桥接 `MjSpec` 元素 → `MjModel` / `MjData`）

使用 `MjSpec` 以编程方式构建模型时，用 `bind()` 将 spec 元素句柄直接映射到编译后的数据：

```python
geom = spec.worldbody.add_geom(name='ball', size=[0.1])
joint = spec.worldbody.add_joint(name='hinge', type=mujoco.mjtJoint.mjJNT_HINGE)
model = spec.compile()
data = mujoco.MjData(model)

# Read through bind
radius = model.bind(geom).size[0]
ball_pos = data.bind(geom).xpos

# Write through bind — always use .set()
data.bind(joint).set('qpos', 0.5)

# Bind lists of spec elements in batch
joints = [spec.joint('j1'), spec.joint('j2')]
joint_angles = data.bind(joints).qpos
```

## 3. 仿真步进与执行流水线

### 步进流水线（`mj_step` 与 `mj_step1` / `mj_step2`）

-   **`mj_step(model, data)`**：将仿真推进一个时间步（`model.opt.timestep`）。
    它在一次完整流程中计算运动学、力、碰撞约束、加速度和数值积分。
-   **拆分步进（`mj_step1` + `mj_step2`）**：当自定义控制器力或外部计算依赖当前运动学状态时使用：

```python
# 1. Advance position kinematics and sensor computations
mujoco.mj_step1(model, data)

# 2. Apply control torques or external forces based on current kinematics
data.ctrl[:] = compute_feedback_control(data.qpos, data.qvel)
data.qfrc_applied[:] = compute_external_forces(data)

# 3. Complete dynamic acceleration and integration
mujoco.mj_step2(model, data)
```

### 传感器与运动学初始化（`mj_forward`）

在零时刻读取传感器或位置之前，调用 `mj_forward()` 而非 `mj_step()`。
`mj_forward` 根据当前状态计算运动学和传感器输出，**不推进仿真时间**：

```python
# ✅ RIGHT — initialize sensors and kinematics at t=0
mujoco.mj_forward(model, data)
initial_obs = data.sensordata.copy()

# Advance physics
for _ in range(num_steps):
  mujoco.mj_step(model, data)
```

### 关键帧重置与状态恢复

使用关键帧可靠地重置配置：

```python
# Python: reset to named keyframe
key_id = model.key('home').id
mujoco.mj_resetDataKeyframe(model, data, key_id)
mujoco.mj_forward(model, data)  # Recompute kinematics and sensors
```

在 C++ 中：
```cpp
int key_id = mj_name2id(m, mjOBJ_KEY, "home");
if (key_id >= 0) {
  mj_resetDataKeyframe(m, d, key_id);
  mj_forward(m, d);
}
```

### 积分器与数值稳定性

通过 `model.opt.integrator` 配置（完整方程及时间步/积分器选择说明见
[computation/index.rst](../../computation/index.rst)，
`.. _geIntegrators:` 和 `.. _geChoosingTimestepIntegrator:`，
[modeling.rst](../../modeling.rst)，`.. _CTuningGlobal:`，以及
[XMLreference.rst](../../XMLreference.rst)，`.. _option-integrator:`）：

-   `mjINT_IMPLICITFAST`（**推荐**）：通过对称 Cholesky 分解实现速度隐式（阻尼、`kv`、
    流体阻力、自由体陀螺力），成本约等于 Euler。
-   `mjINT_IMPLICIT`：增加非对称科里奥利/向心速度导数
    （LU 分解），适用于快速旋转的多连杆链。
-   `mjINT_EULER`（旧版默认）：半隐式 Euler；仅对关节阻尼隐式。
-   `mjINT_RK4`：四阶 Runge-Kutta（每步 4 次求值），适用于平滑、
    能量守恒系统。
-   `mjINT_DISCRETE`（试验中的候选替代方案）：同时对*位置*隐式
    （刚性弹簧、位置伺服 `kp` 及其与接触的耦合；`implicit`/`implicitfast` 仅对速度隐式）。

## 4. 接触与力传感：声明式传感器

### 为什么不应将 `data.contact` 用于观测

`data.contact` 是可变长度缓冲区，每个时间步都会根据活动碰撞对改变大小和顺序：

```python
# ❌ WRONG — variable size and non-deterministic order break ML policies
for c in data.contact:
  if c.geom1 == target_id:
    force = ...
```

### 声明式接触传感器

`<contact>` 传感器（见 [XMLreference.rst](../../XMLreference.rst)，
`.. _sensor-contact:`）以声明方式过滤接触，将其缩减为固定数量的槽位，并将确定性向量直接写入 `data.sensordata`：

```xml
<sensor>
  <!-- Detect contacts between gripper and object: fixed 3 slots -->
  <contact name="grip_contact"
           body1="gripper" body2="object"
           num="3" data="found force normal"
           reduce="maxforce"/>

  <!-- Net wrench on an entire body -->
  <contact name="object_net"
           body1="object"
           data="force torque"
           reduce="netforce"/>
</sensor>
```

在 Python 中读取接触传感器：
```python
contact_data = data.sensor('grip_contact').data  # Always fixed length
```

### 用于标量法向力的触摸传感器

对于标量表面接触力（例如指尖），使用附加到 site 的 `touch` 传感器
（[XMLreference.rst](../../XMLreference.rst)，`.. _sensor-touch:`）。
确保 site 足够大以覆盖 geom：

```python
site = body.add_site(name='fingertip', pos=[0, 0, 0.05], size=[0.02])
spec.add_sensor(
    name='touch_sensor',
    type=mujoco.mjtSensor.mjSENS_TOUCH,
    objname=site.name,
    objtype=mujoco.mjtObj.mjOBJ_SITE,
)
```

## 5. 空间数学（mju_ 库）

MuJoCo 在 C++ 和 Python 中都提供 `mju_` 命名空间下的优化空间数学库。

> [!IMPORTANT]
>
> MuJoCo 四元数为**标量在前**：`[w, x, y, z]`。SciPy 和 ROS 使用
> **标量在后**：`[x, y, z, w]`。转换时需重排分量：
>
> ```python
> # MuJoCo -> SciPy/ROS
> scipy_quat = mj_quat[[1, 2, 3, 0]]
>
> # SciPy/ROS -> MuJoCo
> mj_quat = scipy_quat[[3, 0, 1, 2]]
> ```

### 四