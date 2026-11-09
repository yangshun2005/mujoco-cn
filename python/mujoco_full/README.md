# MuJoCo Python 绑定

[![PyPI Python Version][pypi-versions-badge]][pypi]
[![PyPI version][pypi-badge]][pypi]

[pypi-versions-badge]: https://img.shields.io/pypi/pyversions/mujoco
[pypi-badge]: https://badge.fury.io/py/mujoco.svg
[pypi]: https://pypi.org/project/mujoco/

本包是 [MuJoCo](https://mujoco.org) 物理引擎（[GitHub](https://github.com/google-deepmind/mujoco)）的官方 Python 发行版。这些绑定由 Google DeepMind 开发和维护，并与 MuJoCo 本身的最新进展保持同步。

安装 `mujoco` 会安装：

- **`mujoco-core`**：核心 C API 函数、结构体、常量、枚举、`MjSpec`、经典 OpenGL 渲染（`egl`、`glfw`、`osmesa`）以及经典 `mujoco.viewer`。
- **`mujoco-studio`**：MuJoCo Studio 桌面和 Web 查看器（`mujoco.studio`）以及 Filament 渲染绑定（`mujoco.mjrf`）。
- **`glfw`** 和 **`pyopengl`**：默认 OpenGL 上下文和查看器依赖。

对于不需要 Studio、Filament 或 OpenGL 的最小无头仿真环境（例如 RL 训练或 MJX/Warp worker），可以直接安装 `mujoco-core`（`pip install mujoco-core`），它仅依赖 `numpy`。要将现有的 `mujoco` 安装切换为仅 `mujoco-core`，请运行：

```sh
pip uninstall -y mujoco mujoco-studio && pip install --force-reinstall --no-deps mujoco-core
```

## 安装

推荐通过 [PyPI](https://pypi.org/project/mujoco/) 安装本包：

```sh
pip install mujoco
```

`mujoco-core` 已包含 MuJoCo 库的副本，**无需**单独下载或安装。

## 用法

安装后，可通过 `import mujoco` 导入本包。有关包 API 的更多细节，请参阅我们的[文档](https://mujoco.readthedocs.io/en/stable/python.html)。

我们建议先学习教程 notebook，它涵盖了使用 Python 操作 MuJoCo 的基础知识：
[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/google-deepmind/mujoco/blob/main/python/tutorial.ipynb)

## 版本管理

版本号中的 `major.minor.micro` 部分与绑定所提供的 MuJoCo 版本一致。

## 许可证与免责声明

Copyright 2022 DeepMind Technologies Limited

MuJoCo 及其 Python 绑定采用 Apache License 2.0 许可。您可以在 https://www.apache.org/licenses/LICENSE-2.0 获取许可证副本。

本项目并非 Google 官方支持的产品。