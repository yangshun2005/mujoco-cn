# MuJoCo 版本管理

从 3.5.0 起，MuJoCo 使用自定义语义化版本管理，详见下文。

### 传统语义化版本管理的问题

传统[语义化版本管理](https://semver.org/)的常见定义是：

```
1. MAJOR:  破坏性变更
2. MINOR:  新功能
3. PATCH:  缺陷修复
```

严格遵循此定义，MuJoCo 的大多数发布都算 `MAJOR`。原因是 MuJoCo 的 API 面非常大，尤其是 [MJCF](https://mujoco.readthedocs.io/en/stable/XMLreference.html) 及相关 [mjSpec](https://mujoco.readthedocs.io/en/stable/programming/modeledit.html) API。此外，物理状态积分的数值特性意味着：尽管是确定性的，跨版本的数值可复现性几乎必然[无法保证](https://mujoco.readthedocs.io/en/stable/computation/index.html#reproducibility)。

## 从 3.5.0 起——语义化版本管理

从 3.5.0 起，MuJoCo 使用定义明确的版本管理，语义如下。

### MuJoCo 版本管理语义

```
1. SUPERMAJOR:      破坏性变更和/或重大新功能
2. MAJOR:           破坏性变更，可能包含新功能
3. MINOR_OR_PATCH:  新功能或缺陷修复
```

`MINOR_OR_PATCH` 保证 API 向后兼容，但可能引入也可能不引入新功能；具体请阅读[变更日志](https://mujoco.readthedocs.io/en/stable/changelog.html)，其中会详述变更性质。变更日志也会详述纯 ABI 破坏性变更，例如从公共结构体中移除未使用的属性。此类变更视为 MINOR。如[此处](https://mujoco.readthedocs.io/en/stable/computation/index.html#reproducibility)所述，数值可复现性*从不*保证。

此外：

-   `mj_versionString()` 行为不变，返回如 `"4.2.1"`。
-   `mj_version()` 行为不变，返回整数。对应的头文件常量 `mjVERSION_HEADER` 由各组成部分计算：

    ```
    mjVERSION = (SUPERMAJOR * 1e6) + (MAJOR * 1e3) + MINOR_OR_PATCH
    ```

    例如 4.2.1 为 `4002001`。

## 3.5.0 之前——无语义

3.5.0 之前，版本管理只是一串递增数字，没有明确定义的语义，例外是引入重大新功能的主版本，例如 [3.0.0](https://mujoco.readthedocs.io/en/stable/changelog.html#version-3-0-0-october-18-2023) 引入基于 JAX 的 [MJX](https://mujoco.readthedocs.io/en/stable/mjx.html) 分支。此外：

-   函数 [`const char* mj_versionString()`](https://mujoco.readthedocs.io/en/stable/APIreference/APIfunctions.html#mj-versionstring) 以字符串形式返回版本，例如 `"3.2.7"`。
-   函数 [`int mj_version()`](https://mujoco.readthedocs.io/en/stable/APIreference/APIfunctions.html#mj-version) 返回版本数字拼接后的整数，例如 "3.2.7" 对应 `mjVERSION = 327`。