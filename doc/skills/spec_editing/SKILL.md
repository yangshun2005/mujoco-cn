---
name: mujoco-spec-editing
description: >-
  使用 MjSpec（Python 中的 `mujoco.MjSpec` 和 C++ 中的 `mjSpec`）进行 MuJoCo
  程序化模型编写、编辑与调优：
  刚体、几何体、关节、站点、相机、带前缀命名空间的子模型附加、
  执行器（位置伺服、电机、速度驱动、trntype）、
  基线选项/默认值覆盖（implicitfast、椭圆锥、solref、
  solimp、armature、damping）、碰撞位掩码（contype、conaffinity）、
  摩擦（condim）、默认类（MjsDefault）、程序化网格、删除、
  spec 重新编译（spec.recompile）。仅用于编写或调优 MuJoCo
  MJCF/MjSpec
  模型、添加刚体/几何体/执行器以及附加子模型。
  当未使用 MuJoCo
  `MjSpec` 时，不要用于设计自定义的非 MuJoCo XML/场景图 API、步进仿真（mujoco-python）或渲染
  （mujoco-rendering）。
---

# MuJoCo Spec 编辑：构建与修改模型

> [!TIP]
>
> **官方 MuJoCo 文档（`doc/*.rst`）：** 在 MuJoCo 检出目录中时，直接阅读
> [`../../programming/modeledit.rst`](../../programming/modeledit.rst)
> （`.. _meOverview:`）和 [`../../overview.rst`](../../overview.rst)
> 等短章节；对于
> [`../../modeling.rst`](../../modeling.rst)（`.. _ProceduralModeling:`、
> `.. _Mechanisms:`、`.. _CTuning:`）、
> [`../../XMLreference.rst`](../../XMLreference.rst)、
> [`../../python.rst`](../../python.rst)（`.. _PyModelEdit:`）和
> [`../../APIreference/functions.rst`](../../APIreference/functions.rst)
> 等大型参考文件，用字面量 `.. _Label:` 锚点进行 grep。
> 在检出目录之外，相同的 `doc/*.rst` 目录树会离线随 `pip install mujoco` 发布在
> `pathlib.Path(mujoco.__file__).parent / 'doc'`；否则获取
> `https://mujoco.readthedocs.io/en/latest/<path>.html`（已发布页面用 `/en/stable/`；`functions.rst` 渲染在
> `APIreference/APIfunctions.html` 内，Sphinx 锚点会将标签转为小写，
> 例如 `.. _meOverview:` → `#meoverview`），并在
> `https://github.com/google-deepmind/mujoco/tree/main/` 下浏览仓库路径（`../../../`）。
>
> **相关技能：**
>
> -   [mujoco-python](../python/SKILL.md) — 仿真生命周期、步进和
>     诊断。
> -   [mujoco-rendering](../rendering/SKILL.md) — 离屏图像渲染和
>     相机设置。
> -   [mujoco-studio](../studio/SKILL.md) — 交互式查看器和 Studio
>     插件。

## 1. 使用 MjSpec 进行程序化建模

`MjSpec` 是一个可变的 C++ 和 Python 数据结构，镜像 MJCF XML
Schema，允许以编程方式构建和修改模型。

```python
import mujoco

spec = mujoco.MjSpec()

# Add a body with a freejoint and sphere geom
body = spec.worldbody.add_body(name='ball', pos=[0, 0, 1.0])
body.add_freejoint()
body.add_geom(type=mujoco.mjtGeom.mjGEOM_SPHERE, size=[0.1], rgba=[1, 0, 0, 1])

# Compile to MjModel
model = spec.compile()
```

在 C++ 中：
```cpp
#include <mujoco/mujoco.h>

mjSpec* spec = mj_makeSpec();
mjsBody* body = mjs_addBody(mjs_findBody(spec, "world"), nullptr);
body->pos[2] = 1.0;
mjs_addFreeJoint(body);
mjsGeom* geom = mjs_addGeom(body, nullptr);
geom->type = mjGEOM_SPHERE;
geom->size[0] = 0.1;

mjModel* model = mj_compile(spec, nullptr);
if (!model) printf("%s\n", mjs_getError(spec));
mj_deleteSpec(spec);
```

## 2. 构建层级结构：刚体、几何体、关节和站点

### 添加元素

```python
body = spec.worldbody.add_body(name='arm', pos=[0, 0, 0.5])

# Geom with capsule shape
geom = body.add_geom(
    name='arm_geom',
    type=mujoco.mjtGeom.mjGEOM_CAPSULE,
    size=[0.05, 0.25],
    rgba=[0.2, 0.6, 1.0, 1.0],
)

# Hinge joint
joint = body.add_joint(
    name='shoulder',
    type=mujoco.mjtJoint.mjJNT_HINGE,
    axis=[0, 1, 0],
    range=[-1.57, 1.57],
)

# Sensor site and camera
site = body.add_site(name='grip_site', pos=[0, 0, 0.25])
cam = body.add_camera(name='tool_cam', pos=[0, -0.5, 0.25], xyaxes=[1, 0, 0, 0, 0, 1])
```

### 几何体尺寸参考

| 几何体类型 | `size` 参数语义 |
| :--- | :--- |
| `mjGEOM_SPHERE` | `[radius]` |
| `mjGEOM_CAPSULE` | `[radius, half_length]`（若使用 `fromto` 则为 `[radius]`） |
| `mjGEOM_CYLINDER` | `[radius, half_length]`（若使用 `fromto` 则为 `[radius]`） |
| `mjGEOM_BOX` | `[half_x, half_y, half_z]` |
| `mjGEOM_ELLIPSOID`| `[radius_x, radius_y, radius_z]` |
| `mjGEOM_PLANE` | `[half_x, half_y, grid_spacing]` |

> [!WARNING]
>
> 当为胶囊体或圆柱体指定 `fromto=[x1, y1, z1, x2, y2, z2]` 时，
> `size` 只取**一个值**（`[radius]`）；长度由端点计算得出。

### 朝向关键字

一次只能使用**一种**朝向指定方式：
-   `quat=[w, x, y, z]`（原生表示）
-   `euler=[a1, a2, a3]`（序列由 `spec.compiler.eulerseq` 设定，默认 `'xyz'`）
-   `axisangle=[x, y, z, angle]`
-   `xyaxes=[x_x, x_y, x_z, y_x, y_y, y_z]`
-   `zaxis=[x, y, z]`

`euler` 和 `axisangle` 中的角度单位为度，除非
`spec.compiler.degree = False`（参见 [modeling.rst](../../modeling.rst)，
`.. _COrientation:`，以及 [XMLreference.rst](../../XMLreference.rst)，
`.. _body-geom:`）。

## 3. 子模型组合与附加

通过附加带前缀命名空间的子 spec 来模块化组合模型（参见
[programming/modeledit.rst](../../programming/modeledit.rst)，
`.. _meAttachment:` 和 `.. _meAttributeMerging:`）：

```python
parent = mujoco.MjSpec()
child = mujoco.MjSpec()

child_body = child.worldbody.add_body(name='gripper')
child_body.add_geom(size=[0.05], type=mujoco.mjtGeom.mjGEOM_SPHERE)

# Attach child spec to a mount site on the parent
mount_site = parent.worldbody.add_site(name='mount', pos=[0, 0, 1.0])
parent.attach(child, site=mount_site, prefix='gripper_')
# Child body is renamed to 'gripper_gripper' in parent
```

### 跨 Spec 等式约束

当连接来自两个不同子 spec 的刚体时（例如双臂设置），
先将两个子模型附加到共同的父模型，然后在父 spec 上声明约束：

```python
left_mount = parent.worldbody.add_frame(pos=[0, 0.3, 0])
right_mount = parent.worldbody.add_frame(pos=[0, -0.3, 0])
parent.attach(arm1_spec, frame=left_mount, prefix='left_')
parent.attach(arm2_spec, frame=right_mount, prefix='right_')

eq = parent.add_equality(
    type=mujoco.mjtEq.mjEQ_CONNECT,
    objtype=mujoco.mjtObj.mjOBJ_BODY,
    name1='left_hand',
    name2='right_hand',
)
eq.data[0:3] = [0, 0, 0]  # connect anchor, in name1's body frame
```

> [!NOTE]
>
> `attach()` 要求 `frame=` 或 `site=` 中恰好指定一个；两者都省略会抛出
> `ValueError`。

## 4. 删除与动态修改元素

### 删除元素

将元素对象传给 `spec.delete()`；删除刚体会移除其子树：

```python
spec.delete(spec.body('unwanted_body'))
spec.delete(spec.geom('old_geom'))
spec.delete(spec.joint('redundant_joint'))
spec.delete(spec.actuator('motor'))
```

### 保留状态的重新编译

在运行时更改模型架构时，使用 `spec.recompile` 保留
现有物理状态：

```python
# Add new tool to robot
spec.worldbody.add_geom(name='tool_tip', size=[0.02], type=mujoco.mjtGeom.mjGEOM_SPHERE)

# Recompile keeping positions/velocities intact
model, data = spec.recompile(model, data)
```

## 5. 执行器与伺服

始终显式设置传动类型（`trntype`）：

```python
# 1. Torque Motor
motor = spec.add_actuator(
    name='torque_actuator',
    target='shoulder',
    trntype=mujoco.mjtTrn.mjTRN_JOINT,
)
motor.set_to_motor()

# 2. Stiff Position Servo (Recommended for position control)
servo = spec.add_actuator(
    name='pos_servo',
    target='shoulder',
    trntype=mujoco.mjtTrn.mjTRN_JOINT,
)
servo.set_to_position(kp=200.0, dampratio=1.0)  # Pair with implicitfast (kv defaults to 0)
servo.forcelimited = True
servo.forcerange = [-50.0, 50.0]     # Peak torque in Nm
servo.ctrllimited = True
servo.ctrlrange = [-1.57, 1.57]      # Target angle in radians

# 3. Velocity Actuator
vel = spec.add_actuator(
    name='vel_driver',
    target='wheel_joint',
    trntype=mujoco.mjtTrn.mjTRN_JOINT,
)
vel.set_to_velocity(kv=10.0)
```

> [!CAUTION]
>
> 辅助函数 `set_to_position()` 和 `set_to_motor()` 只配置增益，
> **不**设置传动类型。必须始终指定
> `trntype=mujoco.mjtTrn.mjTRN_JOINT`（或其他有效的传动类型）。参见
> [modeling.rst](../../modeling.rst)（`.. _CActuators:`、`.. _CActShortcuts:`）
> 和 [XMLreference.rst](../../XMLreference.rst)（`.. _actuator:`）。

## 6. 默认类（继承与样式）

默认类（`MjsDefault`）提供级联样式和属性（参见
[modeling.rst](../../modeling.rst)，`.. _CDefault:`，以及
[programming/modeledit.rst](../../programming/modeledit.rst)，`.. _meDefault:`）：

```python
# Create child class under root 'main' class
main_class = spec.default
rubber = spec.add_default('rubber', main_class)
rubber.geom.friction = [1.5, 0.01, 0.001]
rubber.geom.density = 1200.0

# Assign class at creation
body.add_geom(rubber, name='tire', type=mujoco.mjtGeom.mjGEOM_CYLINDER, size=[0.1, 0.05])

# Assign class post-creation (must pass MjsDefault object, NOT a string)
other_geom.classname = rubber
```

### 基线调优覆盖（`<option>` 和根 `<default>`）

若干内部默认值（`Euler`、`impratio=1` 的 `pyramidal` 锥、软
`solref="0.02 1"`、零关节 `armature`/`damping`）是为向后
兼容而保留的，而非为新机器人模型设计。在项目顶部覆盖它们（完整理由和
按优先级排序的参数指南见
[modeling.rst](../../modeling.rst)，`.. _CTuning:`、`.. _CTuningDefaults:`、
`.. _CTuningGlobal:`、`.. _CTuningModel:`）：

```python
spec.option.integrator = mujoco.mjtIntegrator.mjINT_IMPLICITFAST  # DISCRETE for flex elasticity
spec.option.cone = mujoco.mjtCone.mjCONE_ELLIPTIC
spec.option.impratio = 10.0

spec.default.geom.solref = [0.005, 1.0]                 # Keep solref[0] >= 2 * timestep
spec.default.joint.armature = 0.01                      # Reflected rotor inertia (I_rotor * N^2)
spec.default.joint.damping = 0.1                        # Implicit in M + h*D at zero solver cost
```

保留默认 `solimp`：提高 `solimp[0]`（例如提高到 `0.95`）可将静止
穿透和蠕变减半，但会增加求解器迭代次数，因此仅在
`solref` 和椭圆锥之后才考虑使用。

## 7. 碰撞与摩擦调优

### 碰撞过滤位掩码

两个几何体碰撞当且仅当：
`(g1.contype & g2.conaffinity) || (g2.contype & g1.conaffinity)`

```python
# Visual-only geom: never collides; set mass=0 if omitting <inertial> so it
# does not add water-density (1000 kg/m^3) mass (inertiagrouprange defaults to "0 5")
body.add_geom(size=[0.1], contype=0, conaffinity=0, group=1, mass=0)

# Team A collides only with Team B
geom_a = body.add_geom(size=[0.1], contype=1, conaffinity=2)
geom_b = body.add_geom(size=[0.1], contype=2, conaffinity=1)
```

### 接触维度（`condim`）与摩擦

| `condim` | 激活的摩擦分量 | 使用的摩擦索引 |
| :--- | :--- | :--- |
| 1 | 仅法向力（无摩擦） | — |
| 3 | 切向滑动摩擦 | `friction[0]` |
| 4 | 切向 + 扭转（旋转） | `friction[0:2]` |
| 6 | 切向 + 扭转 + 滚动 | `friction[0:3]` |

```python
# High-friction contact for grasping
finger.add_geom(
    type=mujoco.mjtGeom.mjGEOM_CAPSULE,
    size=[0.01, 0.03],
    condim=4,
    friction=[1.2, 0.02, 0.0001],
)
```

关于接触软度、求解器阻抗/参考调优（`solref`、`solimp`、
`priority`/混合），以及父子几何体默认省略碰撞的原因，请阅读
[modeling.rst](../../modeling.rst)（`.. _CTuning:`、`.. _CTuningModel:`、
`.. _CSolver:`、`.. _CContact:`）和 [overview.rst](../../overview.rst)
（`.. _SurprisingCollisions:`、`.. _Soft:`、`.. _Units:`）。

## 8. 程序化网格、材质和传感器

### 程序化网格

```python
vertices = np.array([[0, 0, 0], [1, 0, 0], [0, 1, 0], [0, 0, 1]], dtype=np.float32)
faces = np.array([[0, 2, 1], [0, 1, 3], [0, 3, 2], [1, 2, 3]], dtype=np.int32)
mesh = spec.add_mesh(name='tetra', uservert=vertices.flatten(), userface=faces.flatten())
body.add_geom(type=mujoco.mjtGeom.mjGEOM_MESH, meshname='tetra')
```

凹碰撞网格会被其凸包替换（除非使用 SDF
插件）；优先使用基本体（`sphere`、`capsule`、`box`、`cylinder`），既快又能获得更干净的法线，或者使用
`CoACD` 或