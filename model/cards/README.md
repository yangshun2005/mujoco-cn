# Cards

扑克牌模型，共用 [`assets`](assets) 中的牌面网格和牌面纹理。

<p float="left">
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/cards/cards.xml" title="Open in live.mujoco.org"><img src="https://www.gstatic.com/mujoco/model/cards/cards.png" width="49%"></a>
  <a href="https://live.mujoco.org/?model=github:google-deepmind/mujoco/main/model/cards/house_of_cards.xml" title="Open in live.mujoco.org"><img src="https://www.gstatic.com/mujoco/model/cards/house_of_cards.png" width="49%"></a>
</p>

点击图片可在浏览器查看器 [live.mujoco.org](https://live.mujoco.org) 中打开模型。

| 模型 | 说明 |
| --- | --- |
| [`cards.xml`](cards.xml) | 一副散开的牌。 |
| [`house_of_cards.xml`](house_of_cards.xml) | 一座 26 张牌、四层的纸牌屋。 |

纸牌屋仅靠摩擦力站立，且能保持站立：稳定后，在 2 分钟仿真时间内漂移约 1 毫米。牌厚 0.7 毫米，因此接触属于棘手的一类——近乎平行的面以很小的角度相交，碰撞器能利用的信息最少。模型靠两个细节支撑：

* **碰撞牌是贴合牌真实轮廓的薄盒，并采用过阻尼接触。** 0.7 毫米盒体之间的接触在临界阻尼下会振荡，振荡最终会推倒顶端的牌对；`dampratio` 设为 3 可消除振荡，让纸牌屋以 2 ms 步长运行。可视牌是独立的、不参与碰撞的网格。
* **每张牌都带有零点几度的偏航角，以及沿棱线不到一毫米的抖动。** 完全对称、完美对齐的堆叠对碰撞器最不利：最规整的排列恰恰是会倒塌的那种。

这类摩擦堆叠需要 `cone="elliptic"` 和较大的 `impratio`。加入 `noslip_iterations` 可进一步减少残余蠕变，但会牺牲一些速度；参见文件中的注释。

## 更新日志

* 07-09-2026：新增 `house_of_cards.xml`。