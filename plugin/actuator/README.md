# Actuator 插件

## PID

`mujoco.pid` actuator 插件实现了一个可配置的 [PID 控制器](https://en.wikipedia.org/wiki/Proportional%E2%80%93integral%E2%80%93derivative_controller)：

$$f(t) = K_\text{p} e(t) + K_\text{i} \int_0^t e(\tau) \mathrm{d}\tau + K_\text{d} \frac{\mathrm{d}e(t)}{\mathrm{d}t},$$
其中 $e(t) = u(t) - \ell(t)$ 是控制量 $u$ 与 actuator 长度 $\ell$ 之差。

用法如下：

```xml
<mujoco>
  <extension>
    <plugin plugin="mujoco.pid">
      <instance name="pid">
        <config key="kp" value="40.0"/>
        <config key="ki" value="40"/>
        <config key="kd" value="4"/>
        <config key="slewmax" value="3" />
        <config key="imax" value="1"/>
      </instance>
    </plugin>
  </extension>

  <worldbody>
    <body>
      <joint name="j" type="slide" axis="0 0 1" />
      <geom size="0.01" mass="1"/>
    </body>
  </worldbody>

  <actuator>
    <plugin joint="j" plugin="mujoco.pid" instance="pid" />
  </actuator>
</mujoco>
```

可用选项如下：

|属性 | 默认值 | 含义 |
|----------|---------|---------|
|`kp` | 0 | 控制器的 **P** 增益。 |
|`ki` | 0 | 控制器的 **I** 增益。<p/>若非零，将向 `mjData.act` 添加一个激活变量，包含当前 I 项（单位为力）。 |
|`kd` | 0 | 控制器的 **D** 增益。 |
|`imax` | 可选 | 若指定，I 项产生的力将被裁剪到 `[-imax, imax]` 范围。 |
|`slewmax` | 可选 | PID 控制器设定点可变化的最大速率。<p/>若两个时间步之间请求的变化更大，将被裁剪到 `[ctrl - slewmax * dt, ctrl + slewmax * dt]` 范围。<p/>若指定，将向 `mjData.act` 添加一个激活变量，包含 `ctrl` 的前一值。 |