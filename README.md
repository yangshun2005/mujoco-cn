

> **[中文化] mujoco**
>
> 此项目是 [google-deepmind/mujoco](https://github.com/google-deepmind/mujoco) 的中文翻译版本。
> - 原项目 Stars: 16836
> - 主语言: C++
> - 许可证: Apache-2.0
> - 翻译日期: 2026-11-10
> - 原始 README: [README_en.md](README_en.md)
>
> 说明：下方 README 正文为**原项目文档的中文翻译**，力求忠于原文；本页顶部的「中文快速上手」为**面向中文用户的补充导读**（非原文内容），目的是让你不用读完整篇 README 就能先把项目跑起来。
>
> 如有翻译不准确之处，欢迎提 Issue 或 PR。

---


## 🚀 中文快速上手（补充导读，非原文）

### 这是什么 / 解决什么问题

MuJoCo 全称 **Mu**lti-**Jo**int dynamics with **Co**ntact，是一个通用物理引擎，用于对「多关节结构与其环境交互」进行快速、准确的仿真。它面向机器人学、生物力学、图形与动画、机器学习等需要此类仿真的研究与开发场景。项目由 Google DeepMind 维护，提供 C API，并附带 Python 绑定和 Unity 插件。

### 快速上手（最短路径）

1. **零安装先体验**：打开浏览器访问 [mujoco.org](https://mujoco.org)，或访问 [live.mujoco.org](https://live.mujoco.org) 在 WebAssembly 版 MuJoCo Studio 中查看、拖放自己的模型（比原生 C 慢）。
2. **在本机跑查看器**：按文档中的 Getting Started 指南下载预编译发行版，运行 MuJoCo Studio 应用或经典 `simulate` 查看器。原文未提供具体下载/启动命令，请参考下方 README 正文。
3. **Python 用户安装绑定**（需 Python >= 3.11）：
   ```bash
   pip install mujoco
   ```
   该绑定已自带一份 MuJoCo。
4. **跟着教程走**：在 Google Colab 上打开入门教程 notebook（`python/tutorial.ipynb`），学习 MuJoCo 基础；另有模型编辑、rollout、LQR、最小二乘、MJX、可微物理等教程。

### 环境要求

- **操作系统**：预编译二进制覆盖 Linux（x86-64 与 AArch64）、Windows（仅 x86-64）、macOS（universal）。
- **硬件**：原文未说明额外硬件要求。
- **依赖 / 版本**：
  - Python 绑定要求 Python >= 3.11。
  - 从源码构建需 CMake，最低版本 3.16（见 CMakeLists.txt）。
  - 预编译 Linux wheel 面向 `manylinux2014`，需确认发行版兼容性。
- **国内网络提示**：`pip install mujoco` 从 PyPI 拉取，国内网络可能需要更换镜像源加速。

### 常见坑与注意事项

- **源码构建有稳定性风险**：原文明确提示 `main` 分支最新提交可能不稳定，推荐优先使用预编译发行版。
- **平台限制**：预编译二进制中 Windows 仅支持 x86-64；Linux wheel 面向 `manylinux2014`，部分发行版可能不兼容。
- **浏览器版性能较低**：WebAssembly 版 MuJoCo Studio 比原生 C 慢。
- **版本节奏**：项目目标每月第一周发版，3.5.0 起改用修改版语义化版本，选版本时留意版本号规则变化。

---



---

<h1>
  <a href="https://mujoco.org"><img alt="MuJoCo" src="banner.png" width="100%"/></a>
</h1>

<p>
  <a href="https://mujoco.org" alt="Website">
    <img src="https://img.shields.io/badge/website-mujoco.org-0053d6">
  </a>
  <a href="https://github.com/google-deepmind/mujoco/actions/workflows/build.yml?query=branch%3Amain" alt="GitHub Actions">
    <img src="https://img.shields.io/github/actions/workflow/status/google-deepmind/mujoco/build.yml?branch=main">
  </a>
  <a href="https://mujoco.readthedocs.io/" alt="Documentation">
    <img src="https://readthedocs.org/projects/mujoco/badge/?version=latest">
  </a>
  <a href="https://github.com/google-deepmind/mujoco/blob/main/LICENSE" alt="License">
    <img src="https://img.shields.io/github/license/google-deepmind/mujoco">
  </a>
</p>

**MuJoCo** 代表 **Mu**lti-**Jo**int dynamics with **Co**ntact（带接触的多关节动力学）。它是一个通用物理引擎，旨在促进机器人学、生物力学、图形学与动画、机器学习等领域的研究与开发，这些领域需要对与环境交互的关节结构进行快速而精确的仿真。

本仓库由 [Google DeepMind](https://www.deepmind.com/) 维护。

MuJoCo 提供 C API，面向研究人员和开发者。运行时仿真模块经过调优以最大化性能，运行在内置 XML 编译器预分配的低层数据结构上。该库包含原生桌面窗口和 Web 浏览器中的交互式可视化。MuJoCo 还提供大量用于计算物理相关量的工具函数。

我们还提供 [Python 绑定] 以及 [Unity] 游戏引擎的插件。

## 文档

MuJoCo 的文档位于 [mujoco.readthedocs.io]。下一个发布版本即将推出的功能可在 "latest" 分支的 [更新日志] 中查看。

## 快速开始

1. **在浏览器中试用 MuJoCo。**
访问 [mujoco.org](https://mujoco.org)，无需安装即可在桌面或移动端探索 MuJoCo；或打开 [live.mujoco.org](https://live.mujoco.org)，在 [MuJoCo Studio] 应用的 WebAssembly 版本中查看并拖放你自己的模型（由于运行在 WebAssembly 中，速度比原生 C 慢）。

2. **在本机运行查看器。**
按照文档中的 [快速开始] 指南下载预构建的发布版本，并在本机运行 [MuJoCo Studio] 应用或经典的 `simulate` 查看器（见[此视频](https://www.youtube.com/watch?v=P83tKA1iz2Y)）。

3. **探索我们的在线 IPython notebook。**
如果你是 Python 用户，可以从运行在 Google Colab 上的教程 notebook 开始：

 - **入门**教程讲解 MuJoCo 基础知识：
   [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/google-deepmind/mujoco/blob/main/python/tutorial.ipynb)
 - **模型编辑**教程展示如何以编程方式创建和编辑模型：
   [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/google-deepmind/mujoco/blob/main/python/mjspec.ipynb)
 - **rollout** 教程展示如何使用多线程 `rollout` 模块：
   [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/google-deepmind/mujoco/blob/main/python/rollout.ipynb)
 - **LQR** 教程合成一个线性二次控制器，使一个人形机器人单腿保持平衡：
   [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/google-deepmind/mujoco/blob/main/python/LQR.ipynb)
 - **最小二乘**教程讲解如何使用基于 Python 的非线性最小二乘求解器：
   [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/google-deepmind/mujoco/blob/main/python/least_squares.ipynb)
 - **MJX** 教程提供 [MuJoCo XLA](https://mujoco.readthedocs.io/en/stable/mjx.html)（用 JAX 编写的 MuJoCo 分支）的使用示例：
   [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/google-deepmind/mujoco/blob/main/mjx/tutorial.ipynb)
 - **可微物理**教程利用从 MuJoCo 物理步自动推导的解析梯度训练运动策略：
   [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/google-deepmind/mujoco/blob/main/mjx/training_apg.ipynb)

## 安装

### 预构建二进制文件

带版本号的发布版本以预编译二进制文件的形式提供，可从 GitHub [发布页面] 获取，支持 Linux（x86-64 和 AArch64）、Windows（仅 x86-64）和 macOS（通用）。这是使用本软件的推荐方式。

### 从源码构建

希望从源码构建 MuJoCo 的用户应查阅文档中的[从源码构建]部分。但请注意，`main` 分支顶端的提交可能不稳定。

### Python（>= 3.11）

原生 Python 绑定（已预打包一份 MuJoCo）可通过 [PyPI] 安装：

```bash
pip install mujoco
```

请注意，预构建的 Linux wheel 面向 `manylinux2014`，兼容的发行版见[此处](https://github.com/pypa/manylinux)。有关从源码构建绑定等更多信息，请参阅文档的 [Python 绑定]部分。

## 版本管理

我们计划在每月第一周发布 MuJoCo。我们的版本管理标准在 3.5.0 中改为修改版语义化版本管理，详情见[版本管理](VERSIONING.md)。

## 贡献

我们欢迎社区参与：提问、请求帮助、缺陷报告和功能请求。有关缺陷报告、功能请求及更深入的贡献，请参阅我们的[贡献者指南](CONTRIBUTING.md)和[风格指南](STYLEGUIDE.md)。

## 提问

欢迎在 GitHub 的
["Asking for Help" 讨论区](https://github.com/google-deepmind/mujoco/discussions/categories/asking-for-help)
提问和请求帮助，请聚焦于具体问题。

## 缺陷报告和功能请求

GitHub [Issues](https://github.com/google-deepmind/mujoco/issues) 专用于缺陷报告、功能请求及其他开发相关主题。

## 相关软件
MuJoCo 是众多环境包的底层支撑。下面列出若干绑定和转换工具。

### 绑定

这些包让不同语言的用户能够使用 MuJoCo 功能：

#### 官方绑定：

- [Python 绑定](https://mujoco.readthedocs.io/en/stable/python.html)
  - [dm_control](https://github.com/google-deepmind/dm_control)，Google
    DeepMind 的相关环境栈，包含
    [PyMJCF](https://github.com/google-deepmind/dm_control/blob/main/dm_control/mjcf/README.md)，
    一个用于以编程方式操作 MuJoCo 模型的模块。
- [JavaScript 绑定和 WebAssembly 支持](/wasm/README.md)（受 [stillonearth](https://github.com/stillonearth) 和 [zalo](https://github.com/zalo) 的社区项目启发；[mjswan](https://github.com/ttktjmt/mjswan) 在此基础上扩展了实时策略控制、交互式施力等功能）。
- [C# 绑定和 Unity 插件](https://mujoco.readthedocs.io/en/stable/unity.html)

#### 第三方绑定：

- **MATLAB Simulink**：[Simulink Blockset for MuJoCo Simulator](https://github.com/mathworks-robotics/mujoco-simulink-blockset)
  由 [Manoj Velmurugan](https://github.com/vmanoj1996) 开发。
- **Swift**：[swift-mujoco](https://github.com/liuliu/swift-mujoco)
- **Java**：[mujoco-java](https://github.com/CommonWealthRobotics/mujoco-java)
- **Julia**：[MuJoCo.jl](https://github.com/JamieMair/MuJoCo.jl)
- **Rust**：[MuJoCo-rs](https://github.com/davidhozic/mujoco-rs)

### 转换工具

- **OpenSim**：[MyoConverter](https://github.com/MyoHub/myoconverter) 将
  OpenSim 模型转换为 MJCF。
- **SDFormat**：[gz-mujoco](https://github.com/gazebosim/gz-mujoco/) 是一个
  SDFormat <-> MJCF 双向转换工具。
- **OBJ**：[obj2mjcf](https://github.com/kevinzakka/obj2mjcf)
  一个将复合 OBJ 文件转换为可加载 MJCF 模型的脚本。
- **onshape**：[Onshape to Robot](https://github.com/rhoban/onshape-to-robot)
  将 [onshape](https://www.onshape.com/en/) CAD 装配体转换为 MJCF。

## 引用

如果你在已发表的研究中使用 MuJoCo，请引用：

```
@inproceedings{todorov2012mujoco,
  title={MuJoCo: A physics engine for model-based control},
  author={Todorov, Emanuel and Erez, Tom and Tassa, Yuval},
  booktitle={2012 IEEE/RSJ International Conference on Intelligent Robots and Systems},
  pages={5026--5033},
  year={2012},
  organization={IEEE},
  doi={10.1109/IROS.2012.6386109}
}
```

## 许可证与免责声明

Copyright 2021 DeepMind Technologies Limited.

盒体碰撞代码（[`engine_collision_box.c`](https://github.com/google-deepmind/mujoco/blob/main/src/engine/engine_collision_box.c)）
版权归 Svetoslav Kolev 2016 所有。

`doc` 目录中的 ReStructuredText 文档、图像和视频按 Creative Commons Attribution 4.0 (CC BY 4.0) 许可证条款提供。许可证副本可在
https://creativecommons.org/licenses/by/4.0/legalcode 获取。

源代码按 Apache License, Version 2.0 许可。许可证副本可在 https://www.apache.org/licenses/LICENSE-2.0 获取。

本项目并非 Google 官方支持的产品。

[从源码构建]: https://mujoco.readthedocs.io/en/latest/programming#building-from-source
[快速开始]: https://mujoco.readthedocs.io/en/latest/programming#getting-started
[MuJoCo Studio]: https://mujoco.readthedocs.io/en/latest/studio/
[Unity]: https://unity.com/
[发布页面]: https://github.com/google-deepmind/mujoco/releases
[mujoco.readthedocs.io]: https://mujoco.readthedocs.io
[更新日志]: https://mujoco.readthedocs.io/en/latest/changelog.html
[Python 绑定]: https://mujoco.readthedocs.io/en/stable/python.html#python-bindings
[PyPI]: https://pypi.org/project/mujoco/