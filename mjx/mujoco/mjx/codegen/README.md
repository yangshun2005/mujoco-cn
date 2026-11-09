# MJX Warp 代码生成

读取 `mujoco/mjx/third_party/mujoco_warp/` 中内置的 `mujoco_warp` 源码，在 `mujoco/mjx/warp/` 中生成 MJX-Warp 垫片层。

## 环境准备

在 `mjx/` 根目录下，安装 [`uv`](https://docs.astral.sh/uv/getting-started/installation/) 后，安装最新版 MuJoCo 和本地 MJX：

```bash
uv venv .venv --default-index https://pypi.org/simple
source .venv/bin/activate
uv pip install --upgrade --force-reinstall mujoco --default-index https://pypi.org/simple --extra-index-url https://py.mujoco.org/
uv pip install -e ".[warp,dev]" --default-index https://pypi.org/simple
```

## 运行代码生成

在 `mjx/` 根目录下执行：

```bash
bash mujoco/mjx/codegen/update_for_mujoco_warp.sh
```