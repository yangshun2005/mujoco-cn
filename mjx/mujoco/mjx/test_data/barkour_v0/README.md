# Google Barkour v0

## MJCF 说明

`assets/barkour_v0_mjx.xml` 中的 MuJoCo 配置复制自 https://github.com/deepmind/mujoco_menagerie/google_barkour_v0。针对 brax 对 MJCF 做了以下修改：

* `meshdir` 从 `assets` 改为 `.`。
* 移除了 `frictionloss`。`damping` 改为 0.5239。
* 添加了自定义的 `init_qpos`。
* 为所有足部添加了球形 geom `lowerLegFoot`。关闭了其他所有接触。
* 编译器选项改为 `<option timestep="0.002" iterations="4" solver="CG"/>`。
* 移除了躯干的非视觉 geom，以加快渲染速度。

## 出版物

如果在学术场景中使用本工作，请引用以下出版物：

    @misc{caluwaerts2023barkour,
          title={Barkour: Benchmarking Animal-level Agility with Quadruped Robots},
          author={Ken Caluwaerts and Atil Iscen and J. Chase Kew and Wenhao Yu and Tingnan Zhang and Daniel Freeman and Kuang-Huei Lee and Lisa Lee and Stefano Saliceti and Vincent Zhuang and Nathan Batchelor and Steven Bohez and Federico Casarini and Jose Enrique Chen and Omar Cortes and Erwin Coumans and Adil Dostmohamed and Gabriel Dulac-Arnold and Alejandro Escontrela and Erik Frey and Roland Hafner and Deepali Jain and Bauyrjan Jyenis and Yuheng Kuang and Edward Lee and Linda Luu and Ofir Nachum and Ken Oslund and Jason Powell and Diego Reyes and Francesco Romano and Feresteh Sadeghi and Ron Sloat and Baruch Tabanpour and Daniel Zheng and Michael Neunert and Raia Hadsell and Nicolas Heess and Francesco Nori and Jeff Seto and Carolina Parada and Vikas Sindhwani and Vincent Vanhoucke and Jie Tan},
          year={2023},
          eprint={2305.14654},
          archivePrefix={arXiv},
          primaryClass={cs.RO}
    }