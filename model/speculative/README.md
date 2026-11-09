# 推测性接触

示例模型演示通过推测性接触（`<flag speculative="enable"/>`）实现的连续碰撞检测，可防止快速移动或薄壁刚体在单个时间步内穿透障碍物。底层公式见[推测性接触文档](https://mujoco.readthedocs.io/en/latest/computation/speculative.html)。

<p float="left">
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/speculative/newton.xml" title="在 live.mujoco.org 中打开 newton.xml"><img src="https://www.gstatic.com/mujoco/model/speculative/newton.webp" width="32%"></a>
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/speculative/chain.xml" title="在 live.mujoco.org 中打开 chain.xml"><img src="https://www.gstatic.com/mujoco/model/speculative/chain.webp" width="32%"></a>
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/speculative/flipper.xml" title="在 live.mujoco.org 中打开 flipper.xml"><img src="https://www.gstatic.com/mujoco/model/speculative/flipper.webp" width="32%"></a>
</p>

点击动画可在浏览器查看器 [live.mujoco.org](https://live.mujoco.org) 中打开模型。

| 模型                         | 时间步   | 描述                                                   |
| ---------------------------- | -------- | ------------------------------------------------------ |
| [`newton.xml`](newton.xml)   | 4 ms     | 五块 3 mm 钢板悬挂在双线上                             |
| [`chain.xml`](chain.xml)     | 6 ms     | 由细长矩形胶囊链节组成的自由下落链条                   |
| [`flipper.xml`](flipper.xml) | 10 ms    | 弹簧发射的球被高速铰接球棒击打                         |

在任一模型中禁用 `speculative` 标志（`<flag speculative="disable"/>` 或在 `simulate` UI 中），即可观察到几何体在标称时间步下相互穿透。

## 模型

* **牛顿摆（[`newton.xml`](newton.xml)）：** 五块 3 mm 厚钢板（`size="0.0015 0.08 0.08"`）悬挂在双线上。第一块钢板从 58° 释放，在摆动最低点达到约 3 m/s（每 4 ms 步长移动 12 mm，即钢板厚度的 4 倍）。若无推测性接触，摆动的钢板会直接穿过相邻钢板。
* **互锁链条（[`chain.xml`](chain.xml)）：** 四个自由关节矩形链节由半径 6 mm 的胶囊构成，相邻链节间初始有 13 cm 垂直余量，在 6 ms 时间步下落到固定的水平承接杆上。链条从上到下绷紧时，推测性接触依次使每个链节抵住下一个链节。
* **棒球与球棒（[`flipper.xml`](flipper.xml)）：** 弹簧柱塞将 4 cm 的球射向 1.2 cm 厚的铰接球棒，球棒在刚性扭转弹簧作用下以 10 ms 时间步扫过撞击区。在撞击步内，球的直线行程和球棒的旋转扫掠均超过各自厚度。

## 建模说明

* **弹性恢复（[`newton.xml`](newton.xml)）：** 标准推测性接触将撞击分摊到两步——第一步消除 `(1 - t)` 的法向速度以落到表面，下一步再处理表面接触。当通过直接负 `solref`（使 `h * damping ≈ 2` 在单步内反转法向速度）建模近弹性碰撞时，将刚度设为零（`solref="0 -499"`）并配合高阻抗（`solimp="0.995 0.995 0.001"`），以免推测性位置弹簧项在接近步中抵消弹性阻尼器（注意零刚度不为静止接触提供静态位置弹簧）。
* **单遍预步速度：** 推测性扫掠使用预步速度（`mjData.qvel`），因此被快速邻居撞击的静止中间体只有在求解器运行后才获得撞击后速度。在多体撞击链（如 [`newton.xml`](newton.xml)）中，应在相邻钢板间保留小间隙，或选择单步行程不会一次跨过两个间隙的时间步。

## 变更日志

* 27-10-2026：首次发布。