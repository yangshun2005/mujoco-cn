# 为 MuJoCo 做贡献

我们希望 MuJoCo 成为真正由社区驱动的项目，期待接受你的贡献！

## 贡献之前

### 文档、论坛

请阅读 MuJoCo 的[文档](https://mujoco.readthedocs.io/)，并浏览 GitHub [Issue](https://github.com/google-deepmind/mujoco/issues) 和[讨论](https://github.com/google-deepmind/mujoco/discussions)页面上的当前话题。

### 贡献者许可协议

对本项目的贡献必须附带贡献者许可协议（CLA）。你（或你的雇主）保留贡献的版权；该协议仅授予我们将你的贡献作为项目一部分使用和再分发的许可。请前往 <https://cla.developers.google.com/> 查看你当前的协议存档或签署新协议。

通常只需提交一次 CLA，因此如果你已经提交过（即使是为其他项目），可能无需再次提交。

## 贡献

### 报告 Bug

如何提交一份好的 Bug 报告：

-   使用清晰、描述性的标题。

-   让问题易于复现。如需模型，请将其作为 zip 文件附在 Bug 报告中。复现问题所需的模型和步骤应*最小化*，即移除无关部分。

-   明确说明预期行为。

-   如相关，附上说明性截图。

尽量提供背景信息：

-   如果问题是新出现的，尝试在旧版本中复现。问题最晚在哪个版本中不存在？

-   能否在多个平台上复现该问题？

### 建议增强功能

提交增强建议之前：

-   检查你是否使用的是 MuJoCo 的[最新版本](https://github.com/google-deepmind/mujoco/releases/latest)。

-   快速[搜索](https://github.com/google-deepmind/mujoco/issues)该增强是否已被建议。如果已有，请在现有 Issue 下添加评论，而非新建 Issue。

如何提交一份好的增强建议：

-   使用清晰、描述性的标题。

-   描述当前行为和期望行为。

-   解释该增强为何有用。

-   注明你使用的 MuJoCo 版本和平台/操作系统。

### 贡献代码

-   除小型、直接的 Bug 修复外，请在开始贡献前与我们联系，以便我们提供帮助和可能的指导。提前协调能大大减少后续的挫败感。

-   所有提交都需要审查。请使用 GitHub Pull Request 进行提交。关于 Pull Request 的更多信息，请参阅 [GitHub 帮助](https://help.github.com/articles/about-pull-requests/)。

-   编写测试。MuJoCo 的 C++ 测试使用 [googletest](https://github.com/google/googletest)，Python 绑定代码使用 [absltest](https://abseil.io/docs/python/guides/testing)，Unity 插件中的 C# 代码使用 [nunit](https://nunit.org/)。大多数情况下，Pull Request 只有包含测试才会被接受。MuJoCo 内部代码库目前测试覆盖率不足。如果你想修改未被测试覆盖的函数，需要为现有功能贡献测试，而不仅是你修改的部分。事实上，为现有代码编写测试是开始贡献的好方法。

-   解决编译器警告。

-   所有测试必须在**双精度和单精度**下均通过（带和不带 `mjUSESINGLE`）。编写或更新涉及浮点比较的测试时，请使用 [`test/fixture.h`](test/fixture.h) 中精度感知的工具（如 `MjTol`、`MjNear`），并遵循其中记录的容差校准流程，使用 `MJTOL_SCALE=0`。

-   遵循[风格指南](./STYLEGUIDE.md)。尤其要为代码添加充分的注释。

-   提交小型 Pull Request。我们可能会要求你将大型 Pull Request 拆分为自包含的较小请求，尤其是当 PR 试图实现多个目标时。

-   回应审查者。请对任何问题和评论及时回应。

-   如果你不介意通过 Git 作者身份将贡献归属于你，请在 PR 中告知我们。直接整合变更比走完整的 Pull Request 导入流程要容易得多，这能让你的变更更快合并。

满足所有要求后，你的代码将被合并。感谢你改进 MuJoCo！

### 社区准则

本项目遵循 Google 的[开源社区准则](https://opensource.google/conduct/)。