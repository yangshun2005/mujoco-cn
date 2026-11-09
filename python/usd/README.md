# MuJoCo OpenUSD 绑定（`mujoco-usd`）

[![PyPI version](https://img.shields.io/pypi/v/mujoco-usd)](https://pypi.org/project/mujoco-usd/)

**`mujoco-usd`** 为 [MuJoCo](https://mujoco.org/) 提供 C++ OpenUSD 导入器（`usd_decoder`）、导出器（`mujoco.usd.writer`、`mujoco.usd.layer_sink`）以及自定义物理 Schema（`mujoco.usd.MjcPhysics`、`usdMjcf`）。

```bash
pip install mujoco-usd
```

如需同时安装 `absl-py` 以运行 OpenUSD Python 测试：

```bash
pip install "mujoco-usd[test]"
```

## 版本对齐

`mujoco-usd` 固定使用与其 C++ 扩展和 OpenUSD 插件编译时完全匹配的 `mujoco-core`（`mujoco-core==<version>`）和 `usd-core`（`usd-core==25.11`）版本。