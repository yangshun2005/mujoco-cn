# 弹性机构

受 [Miles Macklin 的 Reduced Elastic Links 实验](https://reports.mmacklin.com/newton-reduced/reduced_elastic_links_implementation.html) 启发，使用 MuJoCo 的多单元二次 flex 和 1D flex 杆实现。几何与 MJCF 均为原创；无需 Newton 源码或外部网格资源。

<p float="left">
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/flex/mechanisms/cantilever.xml" title="在 live.mujoco.org 中打开"><img src="https://www.gstatic.com/mujoco/model/flex/mechanisms/cantilever.png" width="32%"></a>
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/flex/mechanisms/slidercrank.xml" title="在 live.mujoco.org 中打开"><img src="https://www.gstatic.com/mujoco/model/flex/mechanisms/slidercrank.png" width="32%"></a>
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/flex/mechanisms/dipper.xml" title="在 live.mujoco.org 中打开"><img src="https://www.gstatic.com/mujoco/model/flex/mechanisms/dipper.png" width="32%"></a>
</p>

点击图片可在浏览器查看器 [live.mujoco.org](https://live.mujoco.org) 中打开模型。

| 模型 | 机构 | Flex 离散化 | 总自由度 |
| --- | --- | --- | --- |
| [`cantilever.xml`](cantilever.xml) | 三根刚度或阻尼不同的 3D 梁和一根 1D 杆，末端直接连接质量块 | 每根 3D 梁 2 × 1 × 1 个二次单元 | 414 |
| [`slidercrank.xml`](slidercrank.xml) | 曲柄轮通过柔性连杆驱动滑动活塞 | 1D 杆，21 个顶点 | 67 |
| [`dipper.xml`](dipper.xml) | 气缸驱动的柔性臂，带自由摆动的腱悬挂载荷 | 1D 杆，21 个顶点 | 86 |

曲柄滑块电机使用仿射力偏置：力矩 = 50 × (2π/3 − 角速度)。挖掘臂使用相同的速度反馈，目标为 2π rad/s。可见的 7 cm 偏心轮和连杆将其旋转转换为活塞运动，行程约 1 Hz。实际速度随负载略有变化。每个 `crank` 控制量为目标角速度（rad/s）叠加一个偏移量；将其设为负的标称目标值即请求零速度。悬臂梁在重力下运动。

## 悬臂梁对比

四根梁的长度均为 0.9 m，截面 0.07 × 0.055 m，梁质量 0.12 kg，末端质量 0.2 kg。每个末端质量居中于端部截面，无悬挂连杆或偏心载荷。材料参数如下：

| 颜色 / 梁 | 杨氏模量 | 刚度比例 Rayleigh 阻尼系数 |
| --- | --- | --- |
| 青色 / 软 | 8 MPa | 0.002 s |
| 橙色 / 硬 | 16 MPa | 0.002 s |
| 紫色 / 阻尼 | 8 MPa | 0.04 s |
| 黄色 / 杆 | 8 MPa | 0.002 s |

两根软梁的静态平衡相同，但瞬态衰减不同。硬梁弯曲更小、振荡更快。节点关节阻尼为零，以免掩盖指定的材料阻尼。

黄色梁是 1D flex，采用离散弹性杆弯曲，并使用相同截面的 `box` 弹性段，因此按弱（竖直）轴的刚度弯曲。其 24 个顶点通过边等式保持不可伸长。其根部边跨过夹持面并被固定，最后一条边上的两个点等式连接末端质量。其稳定后的末端挠度为 0.0746 m，与 Euler–Bernoulli 预测值 0.0752 m（末端质量加自重）相差不到 1%，而计算成本仅为 3D 梁的一小部分；剩余差距主要来自 8% 挠度的几何非线性，线性梁理论忽略了这一点。边等式设置了自己的 `solref` 和 `solimp`，因为 flex 边约束不使用 `equality` 默认值：在默认边 solref 为 0.02 s 时，轻质杆顶点会使边在负载下拉伸约 2%，杆的下垂比 Euler–Bernoulli 多 4%。

## 连接与悬挂

在 3D 悬臂梁中，精细四面体网格提供可见表面，`cellcount` 控制较粗的变形网格。它们固定根部截面的九个节点。二次单元在弯曲中不会剪切锁定，因此每根梁两个单元即可达到 Euler–Bernoulli 末端挠度的约 10% 以内。

连杆和挖掘臂是 1D 杆，采用离散弹性杆弯曲，质量和 `box` 截面与它们所替代的三线性 3D flex 相同，flex 半径为截面高度的一半以保持视觉粗细，并用边等式保持不可伸长。杆遵循 Euler–Bernoulli 弯曲，而单单元厚度的三线性单元会剪切锁定，刚度远高于其杨氏模量所暗示的值，因此杆使用该模量的 1.8 倍（连杆）和 2.25 倍（挖掘臂）以保持原机构的挠度。其 Rayleigh 阻尼系数除以相同因子，从而保持物理阻尼（系数乘以刚度）不变。它们在机构平面内绕截面的弱轴弯曲，这也是各向同性杆模型使用的刚度。其顶点是世界坐标系中的滑动体，支持离散积分器的稀疏 Newton 求解。

点等式将每个连接点固定到刚性接头或末端质量：3D 端部截面的四个角节点，或杆的两个顶点（杆端的最后两个顶点，以及挖掘臂中支点顶点的两个相邻顶点）。接头可承载普通铰链，如连杆两端和挖掘臂支点处。在挖掘臂中，基座铰链、曲柄驱动的移动杆和末端点连接构成驱动气缸。Flex 自由度为被动。滑动关节具有 30 N·s/m 阻尼，以抵抗活塞运动并加载连杆。

挖掘臂的 0.5 kg 载荷是独立的自由体。一根仅设上限长度的 0.42 m 空间腱将其顶部连接到臂端接头：它传递张力并允许松弛，无需刚性悬挂连杆或载荷姿态控制器。在 1 Hz 时悬挂松弛，载荷大幅翻滚。禁用该腱会使载荷自由下落。

## 变更日志

* 05-10-2026：初始发布。