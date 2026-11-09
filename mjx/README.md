# MuJoCo XLA (MJX)

[![PyPI Python Version][pypi-versions-badge]][pypi]
[![PyPI version][pypi-badge]][pypi]

[pypi-versions-badge]: https://img.shields.io/pypi/pyversions/mujoco-mjx
[pypi-badge]: https://badge.fury.io/py/mujoco-mjx.svg
[pypi]: https://pypi.org/project/mujoco-mjx/

本包是 [MuJoCo 物理引擎](https://github.com/google-deepmind/mujoco) 在 [JAX](https://github.com/jax-ml/jax) 中的重新实现。该库由 Google DeepMind 开发和维护，并与 MuJoCo 本身的最新进展保持同步。

`mujoco-mjx` 包与 MuJoCo API 兼容，但缺少 MuJoCo 中的部分功能。有关功能对等性的更多细节，请参阅我们的[文档](https://mujoco.readthedocs.io/en/stable/mjx.html)。

## 安装

推荐通过 [PyPI](https://pypi.org/project/mujoco-mjx/) 安装本包：

```sh
pip install mujoco-mjx
```

## 使用

安装后，可通过 `from mujoco import mjx` 导入本包。有关该包 API 的更多细节，请参阅我们的[文档](https://mujoco.readthedocs.io/en/stable/mjx.html)。

建议先完成教程 notebook，它介绍了 MJX API，并在几分钟内训练出一个强化学习策略：[![Open In
Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/google-deepmind/mujoco/blob/main/mjx/tutorial.ipynb)

## 版本管理

版本号中的 `major.minor.micro` 部分与本库所对应的 MuJoCo 版本一致。此外，如果我们针对同一 MuJoCo 版本发布 MJX 更新，会追加 `.postN` 后缀，例如 `3.0.1.post2` 表示针对 MuJoCo 3.0.1 的第二次 MJX 更新。

## 许可证与免责声明

Copyright 2023 DeepMind Technologies Limited

MuJoCo 及其库采用 Apache License 2.0 许可。您可以在 https://www.apache.org/licenses/LICENSE-2.0 获取许可证副本。

本项目并非 Google 官方支持的产品。