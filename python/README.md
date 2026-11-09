# MuJoCo 核心 Python 绑定（`mujoco-core`）

[![PyPI Python 版本][pypi-versions-badge]][pypi]
[![PyPI 版本][pypi-badge]][pypi]

[pypi-versions-badge]: https://img.shields.io/pypi/pyversions/mujoco-core
[pypi-badge]: https://badge.fury.io/py/mujoco-core.svg
[pypi]: https://pypi.org/project/mujoco-core/

本包包含 [MuJoCo](https://mujoco.org) 物理引擎（[GitHub](https://github.com/google-deepmind/mujoco)）的核心 Python 绑定（`mujoco-core`）。这些绑定由 Google DeepMind 开发和维护，并与 MuJoCo 本身的最新进展保持同步。

`mujoco-core` 包提供对 MuJoCo 原始 C API 函数、结构体、常量和枚举的直接访问，运行时仅依赖 `numpy`。安装完整的 `mujoco` 包（`pip install mujoco`）时会自动安装它，也可以在最小化的无头仿真环境中单独安装（`pip install mujoco-core`）。要将已有的 `mujoco` 安装切换为仅使用 `mujoco-core`，请运行：

```sh
pip uninstall -y mujoco mujoco-studio && pip install --force-reinstall --no-deps mujoco-core
```

本包的目标不是提供完整的场景/环境/游戏创作 API，因为已有许多包在这方面做得很好。不过，本包确实提供了一些 MuJoCo 本身之外的底层组件，对大多数通过 Python 访问 MuJoCo 的用户来说可能很有用。例如，`egl`、`glfw` 和 `osmesa` 子包包含用于设置 OpenGL 渲染上下文的工具。

## 安装

推荐通过 [PyPI](https://pypi.org/project/mujoco/) 安装本包：

```sh
pip install mujoco
```

包内已附带 MuJoCo 库的副本，**无需**单独下载或安装。

### 源码

**重要：** 只有在修改 Python 绑定（或尝试在极旧的 Linux 系统上运行）时，才需要从源码构建。否则，建议从 PyPI 安装预编译的二进制文件。

如需从源码构建 Python 绑定，请参阅[文档](https://mujoco.readthedocs.io/en/latest/python.html#building-from-source)。

## 用法

安装后，可通过 `import mujoco` 导入本包。有关包 API 的更多细节，请参阅我们的[文档](https://mujoco.readthedocs.io/en/stable/python.html)。

建议先完成教程 notebook，它涵盖了使用 Python 操作 MuJoCo 的基础知识：
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/google-deepmind/mujoco/blob/main/python/tutorial.ipynb)

## 版本管理

版本号中的 `major.minor.micro` 部分与绑定所提供的 MuJoCo 版本一致。此外，如果我们针对同一 MuJoCo 版本发布了 Python 绑定本身的更新，会附加 `.postN` 后缀，例如 `2.1.2.post2` 表示针对 MuJoCo 2.1.2 的绑定的第二次更新。

## 许可证与免责声明

Copyright 2022 DeepMind Technologies Limited

MuJoCo 及其 Python 绑定采用 Apache License 2.0 许可。您可以在 https://www.apache.org/licenses/LICENSE-2.0 获取许可证副本。

这不是 Google 官方支持的产品。