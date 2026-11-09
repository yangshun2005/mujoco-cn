# MuJoCo 风格指南

MuJoCo 代码库遵循内部一致的风格，注重紧凑性和可读性。请在代码贡献中尽可能严格遵循本风格指南。

### 本指南的范围

本指南主要涉及 C/C++ 代码。Python 部分请跳至[下文](#python-代码)。MuJoCo C/C++ 代码分为三大类：

1.  **C 代码：** MuJoCo 核心代码库。包括 `include/` 下的公共头文件，以及 `src/` 下的 C 源文件和内部头文件。本风格指南主要针对此类代码。

2.  **C++ 代码：** MuJoCo C++ 源文件应按照最近的 `.clang-format` 配置（如启用）使用 `clang-format` 格式化。

3.  **Python 及其他语言：** `python/` 下的 Python 文件遵循 Google Python 风格（见[下文](#python-代码)）。`unity/` 下的 C# 文件遵循标准 C# 约定。

### 通用原则

本指南未明确规定的编码风格方面，遵循以下原则：

最大化与其余代码的一致性。 |
----------------------------------------------- |

如果本指南与现有代码存在矛盾，以本指南为准。其他原则包括：

-   遵循[命名约定](https://mujoco.readthedocs.io/en/latest/programming#naming-convention)。
-   节约水平空间：尽量保持行短，尽可能避免换行。
-   慷慨使用垂直空间：代码块之间的空行是好的。
-   保持名称简短。
-   行内注释是代码的一部分，应如此对待。
-   注释和文档使用美式英语。

### C 代码的具体规则

本风格指南将逐步扩展，覆盖 MuJoCo 代码库中 C 编程的大部分方面。在此之前，通常只需查看现有代码并尽量效仿即可。

如果存在 MuJoCo 代码库特有但本指南未提及的一致编码模式，应扩展本指南。如果你发现此类模式，欢迎提交 PR 更新本指南。

#### 缩进

2 空格缩进，使用空格字符而非制表符。

#### 行长度

行长度为 100 字符。在少数情况下，如 [engine_collision_driver.c](https://github.com/google-deepmind/mujoco/blob/c8ff7b3d341560e8cc33fbdcaffbcdbc4c32327c/src/engine/engine_collision_driver.c#L36) 顶部的碰撞表，为可读性允许更长的行。

#### 注释

MuJoCo 大量使用简短的单行注释来描述其下方的代码块。它们被视为代码的重要组成部分。注释应：

-   尽可能简洁，同时保持清晰。
-   前面有空行，除非位于代码块顶部。
-   首字母不大写，不以句号结尾。

关于代码内注释的一个有用启发是：读者仅通过阅读注释就能了解函数中正在发生的事情。

上述第三点的例外是公共头文件中的函数声明注释，它们被视为文档字符串而非代码，因此首字母大写并以句号结尾。这些文档字符串是必需的。

#### 花括号

-   MuJoCo 使用[附着的 K&R 花括号](https://en.wikipedia.org/wiki/Indentation_style#Variant:_mandatory_braces)，包括单行代码块：

    ```c
    // transpose matrix
    void mju_transpose(mjtNum* res, const mjtNum* mat, int nr, int nc) {
      for (int i=0; i < nr; i++) {
        for (int j=0; j < nc; j++) {
          res[j*nr+i] = mat[i*nc+j];
        }
      }
    }
    ```

-   允许无花括号的单行语句，但 `return` 和 `break` 语句除外。此例外的示例请查看 [`mjCModel` 析构函数](https://github.com/google-deepmind/mujoco/search?q=repo%3Adeepmind%2Fmujoco+filename%3Auser_model.cc)。

-   在 `if/else` 块中，当在 `else` 上方插入解释性注释时，允许使用非附着花括号：

    ```c
    // rotate vector by quaternion
    void mju_rotVecQuat(mjtNum res[3], const mjtNum vec[3], const mjtNum quat[4]) {
      // null quat: copy vec
      if (quat[0] == 1 && quat[1] == 0 && quat[2] == 0 && quat[3] == 0) {
        mju_copy3(res, vec);
      }

      // regular processing
      else {
        mjtNum mat[9];
        mju_quat2Mat(mat, quat);
        mju_mulMatVec3(res, mat, vec);
      }
    }
    ```

#### 空格

-   MuJoCo 鼓励在运算符周围合理使用空格以提升可读性。例如下文，注意乘法运算符周围没有空格，而第二和第四个赋值中有对齐空格：

    ```c
    // time-derivative of quaternion, given 3D rotational velocity
    void mju_derivQuat(mjtNum res[4], const mjtNum quat[4], const mjtNum vel[3]) {
      res[0] = 0.5*(-vel[0]*quat[1] - vel[1]*quat[2] - vel[2]*quat[3]);
      res[1] = 0.5*( vel[0]*quat[0] + vel[1]*quat[3] - vel[2]*quat[2]);
      res[2] = 0.5*(-vel[0]*quat[3] + vel[1]*quat[0] + vel[2]*quat[1]);
      res[3] = 0.5*( vel[0]*quat[2] - vel[1]*quat[1] + vel[2]*quat[0]);
    }
    ```

-   比较运算符周围必须有空格。

-   数组下标 `[]` 中的运算符或 `for` 循环中的变量初始化不允许有空格。例如，查看上面的 `mju_transpose` 实现。

-   源文件中函数实现之间需要两个空行。

#### 变量声明

历史上 MuJoCo C 代码库仅使用 C89 风格的变量声明，所有栈变量在函数顶部预先声明。我们正在将代码迁移到 C99 约定，即在尽可能窄的作用域中声明变量。例如，for 循环中的迭代变量大多在窄作用域中声明，如上面的 `mju_transpose` 示例。

新代码应使用 C99 约定。编辑现有函数时，请将现有变量声明移入局部作用域。非常欢迎帮助我们完成迁移的 Pull Request。

### C++ 代码中的错误处理

`mju_error` 并不总是中止进程：Python 绑定和模型编译器安装了错误处理器，会 `longjmp` 回恢复点。`longjmp` 会跳过其展开的栈帧中所有 C++ 对象的析构函数，导致内存泄漏并使互斥锁保持锁定。因此，在 C++ 代码中：

-   当当前或任何调用中的 C++ 栈帧中存在具有非平凡析构函数的对象（`std::string`、`std::vector`、智能指针、锁等）时，不要调用 `mju_error` 或可能调用它的 C 函数。
-   用户输入导致的错误应通过异常或错误码报告，例如编译器中的 `mjCError` 和 XML 解析器中的 `mjXError`。如果公共函数使用 `mju_error` 报告错误，请在 API 边界捕获错误，将消息复制到 `char` 缓冲区，并在所有 C++ 对象销毁后调用 `mju_error`。
-   插件回调、解码器和资源提供者也不应调用 `mju_error`。不要在持有锁时调用它们。
-   测试用户输入可能触及的错误路径。`test/fixture.h` 中的 `MjuErrorMessageFrom` 像 Python 绑定一样安装 `longjmp` 错误处理器；在泄漏检查器下运行测试时会检测到泄漏。

### Python 代码

对于 Python 代码，运行 `pyink foo.py` 以遵循 Google 的 [Python 风格指南](https://google.github.io/styleguide/pyguide.html)。对于导入的排序和清理，运行 `isort foo.py`。`pyink` 和 `isort` 均可通过 `pip install pyink isort` 安装。