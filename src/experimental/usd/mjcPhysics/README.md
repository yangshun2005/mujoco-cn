# mjcPhysics -- 面向 OpenUSD 的 MuJoCo 物理 Schema

本目录包含 `mjcPhysics` 的 USD Schema 定义及配套代码。这些 Schema 支持直接在 USD 中详细指定 MuJoCo 仿真环境。其目标并非取代 UsdPhysics，而是扩展已有概念，仅在必要时创建新类型。

这些 Schema 可以无代码方式使用，也可通过其 C++ 绑定构建。我们已通过 usdGenSchema 为 MuJoCo 内部使用预生成代码，但它同样适用于 MuJoCo 之外原生支持 USD 的应用，如 Omniverse、Houdini 和 Maya。

更多详细信息请参阅[我们的 OpenUSD 文档](https://mujoco.readthedocs.io/en/latest/OpenUSD/index.html)。