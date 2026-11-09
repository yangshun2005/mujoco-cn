# MuJoCo JavaScript 绑定

这是 MuJoCo 物理引擎的官方 JavaScript 和 TypeScript 绑定。

本包提供高层 API，用于与编译为高性能 WebAssembly（WASM）模块的 MuJoCo 核心引擎交互。这些绑定由 Google DeepMind 开发和维护，始终与 MuJoCo 的最新进展保持同步。为简洁起见，下文文档常以"JavaScript"指代，但相关概念同样适用于 TypeScript。

> [!IMPORTANT]
> _这些绑定仍在开发中。详情请参阅[未来工作](#future-work)一节。另请注意，开发主要在 Linux 上使用 Google Chrome 进行。若你在其他操作系统或浏览器上工作，可能会遇到一些问题。我们已在 CI 中成功测试了 macOS 上的绑定，但截至 2025 年 11 月 13 日，Windows 支持仍处于实验阶段（在一台 Windows 11 机器上安装成功，但在其他机器上失败）。_

## 安装

使用 MuJoCo JavaScript 绑定最简单的方式是从 npm 安装 `@mujoco/mujoco` 包：

```sh
npm install @mujoco/mujoco
```

本包为 ESM（`type: module`），包含预编译的 WebAssembly 模块、JavaScript 绑定和 TypeScript 声明。请确保你的打包工具或开发服务器在运行时能提供 `.wasm` 资源。

### 线程模型

`@mujoco/mujoco` 包包含两种不同的引擎构建，以支持不同的浏览器环境和性能需求。

#### 1. 单线程（默认）

标准单线程版本位于包的根目录。它兼容所有现代浏览器，无需特殊安全响应头。
```typescript
import loadMujoco from '@mujoco/mujoco';
```

#### 2. 多线程（MT）

多线程版本位于 `/mt` 子目录。它使用 Web Worker 和 `SharedArrayBuffer` 并行化物理计算。
```typeScript
import loadMujoco from '@mujoco/mujoco/mt';
```

> [!NOTE]
> 由于使用了 `SharedArrayBuffer`，浏览器需要跨源隔离（Cross-Origin Isolation）才能启用多线程。你的 Web 服务器必须发送以下 HTTP 响应头：
>    - `Cross-Origin-Opener-Policy: same-origin`
>    - `Cross-Origin-Embedder-Policy: require-corp`
>
> 若缺少这些响应头，模块将初始化失败。

## 从源码构建

### 前置条件

> [!NOTE]
> 请从顶层目录运行本 README 中的所有命令。

- 编译 [`bindings.cc`](codegen/generated/bindings.cc) 文件（该文件生成 `.wasm` WebAssembly 文件、`.js` JavaScript 导入和 `.d.ts` TypeScript 声明文件）需要 Emscripten SDK 版本 `4.0.10`。更高版本可能可用，但未经测试。设置 SDK 的方法如下，你可以在任意位置运行，但本 README 中其余命令仅在你 source 了 `emsdk_env.sh` 脚本的 shell 中有效。

  ```sh
  git clone https://github.com/emscripten-core/emsdk.git
  ./emsdk/emsdk install 4.0.10
  ./emsdk/emsdk activate 4.0.10
  source ./emsdk/emsdk_env.sh
  ```

- 运行 JavaScript 测试和演示应用需要 `node` 和 `npm`。我们建议使用 [nvm](https://github.com/nvm-sh/nvm) 管理它们。测试、演示和绑定构建过程还需要各种 JavaScript 依赖。这些依赖应位于 `wasm` 文件夹中。安装它们并确保后续命令能找到它们，请运行：

  ```sh
  npm install --prefix ./wasm
  export PATH="$(pwd)/wasm/node_modules/.bin:$PATH"
  ```

- 修改绑定需要 `python3`，因为 [`bindings.cc`](codegen/generated/bindings.cc) 文件由 Python 脚本生成。运行绑定生成器测试需要 `absl`，`pytest` 会很有帮助。按如下方式设置包含这些依赖的 Python 环境：

  ```sh
  python3 -m venv .venv
  source .venv/bin/activate
  pip install -r python/build_requirements.txt
  ```

> [!TIP]
> _Emscripten 文档完善。我们建议阅读 [Emscripten 编译器设置](https://emscripten.org/docs/tools_reference/settings_reference.html)、[Emscripten SDK](https://emscripten.org/docs/tools_reference/emsdk.html) 和 [Embind](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/embind.html) 库相关章节。要了解以浏览器为平台的限制和注意事项，请参阅[移植](https://emscripten.org/docs/porting/index.html#porting)一节。_

[`bindings.cc`](codegen/generated/bindings.cc) 文件编译后生成 `.wasm` WebAssembly 文件、`.js` JavaScript 导入和 `.d.ts` TypeScript 声明文件。这些文件用于从 JavaScript 调用 MuJoCo。生成它们前，请确保 npm 和 Emscripten SDK 前置条件已设置，然后运行：

```sh
emcmake cmake -B build && cmake --build build
```

该命令将在项目根目录下生成以下文件夹：

- `build`：包含使用 Emscripten 编译的 MuJoCo。
- `wasm/dist`：包含 WebAssembly 模块、`.js` 和 `.d.ts` 文件。

这些文件夹中的资源编译为单线程运行。若需使用多线程版本，请传入 `-DMUJOCO_WASM_THREADS=ON` 标志：

```sh
emcmake cmake -B build -DMUJOCO_WASM_THREADS=ON && cmake --build build
```

### 示例应用

生成绑定后，即可使用 MuJoCo 编写 Web 应用。我们提供了一个使用 Three.js 渲染简单仿真的基础 Web 应用，运行以下命令试用：

```sh
npm run dev:demo --prefix ./wasm
```

你可能更倾向于用 C++ 编写整个应用并使用 Emscripten 编译。若如此，则无需使用这些绑定，因为你只需编写极少量 JavaScript，而这些绑定的粒度可能不合适（例如，你可能希望在 `requestAnimationFrame` 调用的 C++ 回调中调用多个 MuJoCo 函数）。

我们还发现混合方式很有帮助，因为直接在 JavaScript 中使用浏览器 API 通常更方便。若你选择用 C++ 编写应用并使用 Emscripten 编译，可将 `bindings.cc` 中 `EMSCRIPTEN_BINDINGS` 的子集复制到应用源文件中。

## 用户指南
### 命名访问

绑定支持命名访问方法，类似 Python 绑定，可按名称或索引便捷访问模型和数据元素。例如，可通过 `model.geom('mygeom')` 按名称访问几何体，或通过 `data.jnt('myjoint')` 访问关节。

有关命名访问的更多细节和示例，请参阅[命名访问测试](tests/bindings_test.ts#L1876-L2378)和[文档](https://mujoco.readthedocs.io/en/stable/python.html#named-access)。

### 内存管理
Embind 包装的 C++ 对象句柄（在 JavaScript 中创建或返回）位于 WebAssembly 堆上，**不会**被 JS 运行时垃圾回收。

任何暴露给 JS 的堆分配 C++ 对象（例如通过 `new Module.MyClass(...)` 创建，或作为指针/引用从绑定返回）在不再需要时必须显式释放，以避免内存泄漏。

使用包装实例上生成的 `.delete()` 方法销毁底层 C++ 对象：

```typescript
const obj = new Module.MyClass(...);
// ... 使用 obj ...
obj.delete(); // 释放 C++ 内存
```

注意每个创建的对象只能调用一次 `.delete()`（重复删除会报错）。在可能抛出异常或提前返回的 JS 代码路径中，确保在 finally 块中执行删除，或封装生命周期管理以避免泄漏。

> [!IMPORTANT]
> _Embind 文档强烈建议 JavaScript 代码显式删除其收到的任何 C++ 对象句柄。_

### 复制与引用

通过 WASM 绑定与 MuJoCo 对象交互时，理解数据访问方式很重要。`MjModel` 和 `MjData` 等对象的属性有两种数据暴露方式：按复制或按引用。

#### 1. 按复制（基于值的访问）

某些属性返回访问时刻数据的副本。这常见于需要从 C++ 编组到 JavaScript 的复杂数据结构。

一个关键例子是 `MjData.contact`。访问 `data.contact` 时，你会得到一个包含仿真该特定时刻接触点副本的对象。

若推进仿真，它们不会更新。必须再次访问 `data.contact` 才能获取新的接触信息。

该对象是使用 `register_vector<T>` 暴露 `std::vector` 时由 [Emscripten 的 Embind 库](https://emscripten.org/docs/porting/connecting_cpp_and_javascript/embind.html#built-in-type-conversions)生成的 JavaScript 代理接口。它本质上是一个"桥接"对象。

```typescript
export interface MjContactVec extends ClassHandle {
  /** 在向量末尾追加新元素，长度加一。 */
  push_back(_0: MjContact): void;
  /** 将向量调整为指定元素数量，新槽位用提供的值填充。 */
  resize(_0: number, _1: MjContact): void;
  /** 返回向量当前存储的元素总数。 */
  size(): number;
  /** 获取指定索引处的元素，索引越界时返回 undefined。 */
  get(_0: number): MjContact | undefined;
  /** 覆盖指定索引处的元素；成功返回 true，索引无效返回 false。 */
  set(_0: number, _1: MjContact): boolean;
}
```

示例：
```typescript
// 获取当前时刻的接触点。
const contacts = data.contact;

// 推进仿真
mujoco.mj_step(model, data);

// `contacts` 现已过期。要获取新接触点，必须再次访问该属性：
const newContacts = data.contact;

// 不再需要时记得删除所有创建的对象。
contacts.delete();
newContacts.delete();
```

#### 2. 按引用（基于视图的访问）

许多属性（尤其是大型数值数组）返回直接指向 WebAssembly 内存的实时视图。这种方式非常高效，避免了复制大量数据。

一个关键例子是 `MjData.qpos`（关节位置）。获取该数组的引用时，它直接指向仿真状态数据。仿真中的任何变化（例如调用 `mj_step` 后）都会立即反映在该数组中。

```typescript
// `qpos` 是仿真状态的实时视图。
const qpos = data.qpos;

console.log(qpos[0]); // 打印初始位置

// 推进仿真
mujoco.mj_step(model, data);

// `qpos` 自动更新。
console.log(qpos[0]); // 打印新位置

// 不再需要时记得删除所有创建的对象。
data.delete();
```

### 数据布局：行主序矩阵

当 MuJoCo C API 的函数返回矩阵（或需要矩阵作为输入）时，JavaScript 绑定将其表示为扁平的一维 `TypedArray`。元素按行主序存储。

例如，3x10 矩阵将返回为包含 30 个元素的扁平数组。前 10 个元素为第一行，接下来 10 个为第二行，依此类推。

示例：访问 `(row, col)` 处的元素
```typescript
// 以扁平数组存储的 3x10 矩阵。
const matrix: Float64Array = ...;
const nRows = 3;
const nCols = 10;

// 访问第 `i` 行第 `j` 列的元素：
const element = matrix[i * nCols + j];
```

### 使用输出参数

MuJoCo C API 中许多函数使用"输出参数"返回数据。即它们不返回值，而是将结果写入通过引用（使用指针）传入的某个参数。在 JavaScript 绑定中，需专门处理这些情况。

你会遇到两种主要场景：

#### 1. 类数组输出参数

当函数期望指向基本类型的指针（如 `mjtNum*` 或 `int*`）以写入数组值时，需在 JavaScript 侧为结果预分配内存。我们提供了辅助类：`mujoco.Uint8Buffer`、`mujoco.DoubleBuffer`、`mujoco.FloatBuffer` 和 `mujoco.IntBuffer`。

使用方法如下：

1.  创建缓冲区：用正确大小的初始数组（如全零数组）实例化相应的缓冲区类。
2.  调用函数：将缓冲区实例作为输出参数传给函数。
3.  访问结果：使用缓冲区的 `.getView()` 方法获取 C++ 函数写入数据的 `TypedArray` 视图。
4.  释放内存：使用完缓冲区后，必须调用 `.delete()` 方法释放底层内存，防止内存泄漏。

示例：旋转向量

函数 `mju_rotVecQuat` 用四元数 `quat` 旋转向量 `vec`，并将结果存入输出参数 `res`。

```typescript
// 创建缓冲区存放 3D 向量结果。
const res = new mujoco.DoubleBuffer([0, 0, 0]);

const vec = [1, 0, 0];
const quat = [0.707, 0, 0, 0.707]; // 绕 z 轴旋转 90 度

try {
  // 将缓冲区作为输出参数调用函数。
  mujoco.mju_rotVecQuat(res, vec, quat);

  // 以 Float64Array 获取结果。
  const resultView = res.getView();
  console.log(resultView); // 预期：约 [0, 1, 0]

} finally {
  // 重要：释放为缓冲区分配的内存。
  res.delete();
}
```

#### 2. 结构体输出参数（如 mjvCamera*、mjvScene*）

当函数修改通过指针传入的结构体时，应传入相应 JavaScript 包装类的实例。底层 C++ 结构体将就地修改。

示例：更新场景

函数 `mjv_updateScene` 用 `mjModel` 和 `mjData` 的信息填充 `mjvScene` 对象。
```typescript
// 创建所需结构体的实例。
const model = mujoco.MjModel.from_xml_string(xmlContent);
const data = new mujoco.MjData(model);
const scene = new mujoco.MjvScene(model, 1000);
const option = new mujoco.MjvOption();
const perturb = new mujoco.MjvPerturb();
const camera = new mujoco.MjvCamera();

// ...（推进仿真等）

// 更新场景。'scene' 对象由函数修改。
mujoco.mjv_updateScene(
    model,
    data,
    option,
    perturb,
    camera,
    mujoco.mjtCatBit.mjCAT_ALL.value,
    scene
);

console.log('场景中的几何体数量:', scene.ngeom);

// 不再需要时记得删除所有创建的对象。
scene.delete();
camera.delete();
perturb.delete();
option.delete();
data.delete();
model.delete();
```

与缓冲区一样，你负责管理这些结构体实例的内存，完成后必须调用 `.delete()`。

### 枚举
通过 `.value` 访问：
```javascript
mujoco.mjtDisableBit.mjDSBL_CLAMPCTRL.value
```

### 常量
标量常量可作为属性访问：

```javascript
mujoco.mjNEQDATA
```

非标量常量（如 `mjFRAMESTRING`）同样作为属性访问，返回 JavaScript 数组：

```javascript
mujoco.mjFRAMESTRING
```

这将返回 MuJoCo `mjFRAMESTRING` 中值的 JavaScript 数组表示。

> [!NOTE]
> 你会注意到 `mjFRAMESTRING` 等常量的类型为 `any`。这是因为它们在 C++ 中使用 `emscripten::val::array()` 绑定，而 Embind 在 TypeScript 定义文件中将 `emscripten::val` 映射为 `any`。虽然可用 `EMSCRIPTEN_DECLARE_VAL_TYPE(StringArray)` 将 `StringArray` 定义为 `emscripten::val` 的别名，并提示 Embind 如何处理函数签名中的转换或使用 `.as()`，但这不会改变 `emscripten::constant` 对属性类型的推断方式。

## 开发

修改绑定需修改 [`bindings.cc`](codegen/generated/bindings.cc) 文件，但不应手动修改。该文件由 [`codegen`](codegen) 文件夹中的 Python 脚本和模板文件生成。要编辑绑定，需修改这些文件并使用以下命令重新生成 [`bindings.cc`](codegen/generated/bindings.cc)：

```sh
PYTHONPATH=python/mujoco python3 -m wasm.codegen.update
```

代码生成脚本使用 MuJoCo 的 Python introspect 库生成 Embind `EMSCRIPTEN_BINDINGS` 块，将 C++ 函数和类绑定到 JavaScript。绑定的函数和类是 MuJoCo C API 的包装器。这些包装器为添加边界检查和友好错误报告等功能提供了便利位置。

### 测试

1. **JavaScript API 测试。**
   验证各种 MuJoCo 函数和类从 JavaScript 调用时正常工作。运行测试：

   ```sh
   npm run test --prefix ./wasm
   ```

2. **JavaScript API 基准测试。**
   当前基准测试检查 JavaScript/C++ 共享内存缓冲区的性能。我们将逐步扩大基准测试覆盖范围。运行基准测试：

   ```sh
   npm run benchmark --prefix ./wasm
   ```

3. **绑定生成器测试。**
   这些测试在开发或扩展绑定时相关。以下命令查找并运行 `wasm` 文件夹中所有 `test_*.py` 或 `*_test.py` 文件：

   ```sh
   PYTHONPATH=python/mujoco python3 -m pytest ./wasm
   ```

### 调试

我们提供了一个"沙盒"应用，可快速编写代码在浏览器中运行。在 [`main.ts`](tests/sandbox/main.ts) 文件中编写代码，并使用以下命令在浏览器中执行：

```sh
npm run dev:sandbox --prefix ./wasm
```

页面将显示为空白，因为脚本仅输出到控制台。你可在指定占位符处添加代码，并使用 Chrome DevTools 调试。可以设置调试工作流，使跨语言