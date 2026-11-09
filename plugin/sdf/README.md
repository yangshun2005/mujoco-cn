<h1>
  <a href="#"><img alt="MuJoCo" src="../../banner.png" width="100%"/></a>
</h1>

## 有符号距离函数（SDF）插件

这些是实现隐式几何的第一方插件，使用 SDF。它们可应用于 **geom** 和 **mesh**（位于 **asset** 部分）。示例模型见[此文件夹](../../model/plugin/sdf/)。

### Bolt

实现在 [bolt.cc](bolt.cc) 中。用法示例见 [nutbolt.xml](../../model/plugin/sdf/nutbolt.xml)。

此插件实现带六角头的螺栓，类似 https://www.shadertoy.com/view/XtffzX。

参数：

   - `radius` [m]：螺栓半径（默认 `0.26`）。

### Bowl

实现在 [bowl.cc](bowl.cc) 中。用法示例见 [bowl.xml](../../model/plugin/sdf/bowl.xml)。

此插件实现切割后的空心球体，来自 https://www.shadertoy.com/view/7tVXRt。

参数：

   - `height` [m]：切割平面位置（默认 `0.4`）。
   - `radius` [m]：球体半径（默认 `1`）。
   - `thickness` [m]：碗壁厚度（默认 `0.02`）。

### Gear

实现在 [gear.cc](gear.cc) 中。用法示例见 [gear.xml](../../model/plugin/sdf/gear.xml)。

此插件实现 2D 齿轮几何的 3D 拉伸，来自 https://www.shadertoy.com/view/3lG3WR。

参数：

   - `alpha` [m]：齿轮初始旋转角（默认 `0`）。
   - `diameter` [m]：齿轮直径（默认 `2.8`）。
   - `teeth` []：齿数（默认 `25`）。

### Nut

实现在 [nut.cc](nut.cc) 中。用法示例见 [nutbolt.xml](../../model/plugin/sdf/nutbolt.xml)。

此插件实现六角螺母，与 https://www.shadertoy.com/view/XtffzX 中的螺栓头相同。

参数：

   - `radius` [m]：螺母半径（默认 `0.26`）。

### Torus

实现在 [torus.cc](torus.cc) 中。用法示例见 [torus.xml](../../model/plugin/sdf/torus.xml)。

此插件实现圆环体。

参数：

  - `radius1` [m]：主半径（默认 `0.35`）。
  - `radius1` [m]：次半径（默认 `0.15`）。

### 如何编写自己的 SDF

在本 README 所在的 SDF 文件夹中创建 `MySDF.h` 和 `MySDF.cc` 文件。使用以下接口实现你的 SDF：

```
struct MySDFAttribute {
  static constexpr int nattribute =
  /* insert the number of attributes */;
  static constexpr char const* names[nattribute] =
  /* an array of attributes with the same order as the attribute array in your SDF class */;
  static constexpr mjtNum defaults[nattribute] =
  /* an array of default values for your attributes */;
};

class MySDF {
 public:
  // creates a new MySDF instance or returns null on failure.
  static std::optional<MySDF> Create(const mjModel* m, mjData* d, int instance);
  MySDF(MySDF&&) = default;
  ~MySDF() = default;

  // functions that return the SDF and its gradient at a query point
  mjtNum Distance(const mjtNum point[3]) const;
  void Gradient(mjtNum grad[3], const mjtNum point[3]) const;

  // a call to this needs to be added to register.cc
  static void RegisterPlugin();

  // an array of attributes with the same order as in the struct above
  mjtNum attribute[MySDFAttribute::nattribute];

 private:
  MySDF(const mjModel* m, mjData* d, int instance);
};
```