# 弹塑性

六个模型，演示关节弹簧和肌腱弹簧（`springyield`）以及 2D 壳和 3D 连续体 flex（`yield`）中的永久塑性变形。除 [`clay.xml`](clay.xml) 外，每个场景都在相同载荷下将一个弹性参考（`yield = 0`）放在弹塑性对应体旁边。塑性状态保存在 `mjData.plastic` 中，因此重置仿真会恢复未变形的参考几何。

<p float="left">
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/plastic/wire.xml" title="在 live.mujoco.org 中打开 wire.xml"><img src="https://www.gstatic.com/mujoco/model/plastic/wire_v2.webp" width="32%"></a>
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/plastic/paper.xml" title="在 live.mujoco.org 中打开 paper.xml"><img src="https://www.gstatic.com/mujoco/model/plastic/paper_v2.webp" width="32%"></a>
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/plastic/foil.xml" title="在 live.mujoco.org 中打开 foil.xml"><img src="https://www.gstatic.com/mujoco/model/plastic/foil.webp" width="32%"></a>
</p>
<p float="left">
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/plastic/clay.xml" title="在 live.mujoco.org 中打开 clay.xml"><img src="https://www.gstatic.com/mujoco/model/plastic/clay.webp" width="32%"></a>
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/plastic/roller.xml" title="在 live.mujoco.org 中打开 roller.xml"><img src="https://www.gstatic.com/mujoco/model/plastic/roller_v2.webp" width="32%"></a>
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/plastic/torsion.xml" title="在 live.mujoco.org 中打开 torsion.xml"><img src="https://www.gstatic.com/mujoco/model/plastic/torsion_v2.webp" width="32%"></a>
</p>

点击动画可在浏览器查看器 [live.mujoco.org](https://live.mujoco.org) 中打开模型。

| 模型 | 公式 | 屈服参数 | 自由度 | 塑性状态 | 时间步长 |
| --- | --- | --- | --- | --- | --- |
| [`wire.xml`](wire.xml) | 铰链关节与空间肌腱 | `springyield="2.5"`、`"0.5"`（关节）；`"0.085"`、`"0.042"`（肌腱） | 54 | 28 | 2 ms |
| [`paper.xml`](paper.xml) | 2D 壳（`elastic2d="bend"`，边等式约束） | `yield="0.00001"` | 1736 | 800 | 4 ms |
| [`foil.xml`](foil.xml) | 2D 壳（`elastic2d="both"`） | `yield="0.00016 0.0016"`（拉伸、弯曲） | 792 | 1320 | 3 ms |
| [`clay.xml`](clay.xml) | 3D 四面体网格（23 × 23 × 5） | `yield="0.0030"` | 6349 | 69696 | 3 ms |
| [`roller.xml`](roller.xml) | 3D 三线性单元（4 × 6 × 2 对比 6 × 10 × 2） | `yield="0.003"` | 794 | 720 | 3 ms |
| [`torsion.xml`](torsion.xml) | 3D 三线性单元（2 × 2 × 8） | `yield="0.025"` | 438 | 192 | 3 ms |

## 模型

* **1D 关节与肌腱弹簧（[`wire.xml`](wire.xml)）：** 三根 12 段悬臂线材（弹簧钢、黄铜、软铜）和三根对应的悬索，由下落球体加载，球体撞击后滚开。当弹簧挠度超过 `springyield` 时，静止构型发生偏移，将弹力钳制在屈服极限。
* **仅弯曲不可伸长的壳（[`paper.xml`](paper.xml)）：** 两张 17 × 17 薄片由中心穿过黄铜环拉动（`solver="Newton"`）。`<edge equality="true"/>` 保持膜边缘不可伸长，`elastic2d="bend"` 提供弯曲刚度；弹性布料脱离环后展开，而纸巾保留径向折痕。
* **拉伸与弯曲分别屈服（[`foil.xml`](foil.xml)）：** 三个钢球落到倾斜的 13 × 13 薄片上（`elastic2d="both"`）。`yield="0.00016 0.0016"` 中的两个阈值使膜拉伸的屈服应变比弯曲小 10 倍，因此撞击留下清晰的凹坑，而不会出现弯曲漂移失控。两盏非阴影聚光灯置于默认相机的镜像位置，投射的高光在每个凹痕周围发生畸变。
* **3D 连续体冲压（[`clay.xml`](clay.xml)）：** 弹簧加载的铰链压机将十字（`+`）模具压入底部锚定的 23 × 23 × 5 四面体块。体积–偏量（von Mises J₂）返回映射在塑性剪切下保持体积，压机回弹后留下永久压痕。
* **齿状冲压滚轮（[`roller.xml`](roller.xml)）：** 一根滚轴携带两个四齿轮，横穿弹性三线性板（`4 × 6 × 2` 单元）和塑性三线性板（`6 × 10 × 2` 单元），在塑性块上压出间隔的凹痕，而弹性块回弹平整。
* **扭转与释放（[`torsion.xml`](torsion.xml)）：** 偏心擒纵扳手将两根 `2 × 2 × 8` 三线性杆扭转超过 180° 后从驱动销上滑脱，塑性杆留下约 82° 的永久扭转，而弹性杆回扭至零。

## 建模说明

* **带细分表皮的三线性单元：** 插值 flex（`dof="trilinear"`）将仿真自由度（`cellcount`）与视觉和碰撞表面网格（`count`）解耦。在 [`roller.xml`](roller.xml) 和 [`torsion.xml`](torsion.xml) 中设置 `count = 2 * cellcount + 1`，可在零自由度开销下平滑表面着色和接触几何。

## 变更日志

* 2026-04-11：首次发布。