# 干砌石拱

三座无砂浆砌体拱，仅靠重力和摩擦力支撑。每座拱都是一条自由悬浮的拱石链，搁在静态拱座上：去掉摩擦力它们就会坍塌，因此这是对求解器能否忠实解析静止状态下大型摩擦接触网络的直接检验。

<p float="left">
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/arch/roman.xml" title="在 live.mujoco.org 中打开"><img src="https://www.gstatic.com/mujoco/model/arch/roman.png" width="32%"></a>
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/arch/gothic.xml" title="在 live.mujoco.org 中打开"><img src="https://www.gstatic.com/mujoco/model/arch/gothic.png" width="32%"></a>
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/arch/hyperbolic.xml" title="在 live.mujoco.org 中打开"><img src="https://www.gstatic.com/mujoco/model/arch/hyperbolic.png" width="32%"></a>
</p>

点击图片可在浏览器查看器 [live.mujoco.org](https://live.mujoco.org) 中打开模型。

| 模型 | 拱型 | 拱石数 | 时间步长 |
| --- | --- | --- | --- |
| [`roman.xml`](roman.xml) | 半圆 | 27 | 6 ms |
| [`gothic.xml`](gothic.xml) | 等边尖拱 | 19 | 8 ms |
| [`hyperbolic.xml`](hyperbolic.xml) | 加权悬链线 | 29 | 8 ms |

三者均在 `integrator="discrete"` 下运行，该积分器在这些时间步长下遵循建模时设定的接触刚度，且每个模型都以静止状态加载：拱石位姿即拱在载荷下稳定后的形态。放任不管，它们五分钟内会漂移一两毫米；罗马拱的接缝在起拱点附近接近竖直，同期还会下移约一厘米。双曲拱是索状拱：其中心线是自身锥形自重的加权悬链线，因此只承受纯压力，完全不蠕变。

罗马拱和哥特拱的拱石是普通楔形，经收缩使相邻石块间距 0.05 mm。双曲拱最为精细，对它而言有两个建模选择比任何求解器设置都更重要；两者都归结为绝不给碰撞器留下可断裂的绑定：

* **接缝为大面贴小面。** 拱石沿拱的深度方向逐渐收窄，因此每个接缝都有明确的归属面，而非两个尺寸相同的重合面。
* **下表面沿径向内缩。** 若不内缩，内弧面和外弧面边缘会在摇摆平面内边对边相接，激发摇摆模态；内缩后，振铃能量下降一个数量级以上。

此类摩擦堆叠需要 `cone="elliptic"` 和较大的 `impratio`；这些拱使用 10–100。具体配方见各文件中的注释。

## 变更日志

* 07-09-2026：首次发布。