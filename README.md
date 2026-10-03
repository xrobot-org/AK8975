# AK8975

AK8975 三轴磁力计（SPI）驱动模块 / Driver Module for the AK8975 3-axis magnetometer over SPI

## 1. 模块作用 / Purpose

构造时，AK8975 配置 SPI（时钟极性高、第二个边沿采样、分频 DIV_4），读取 `WIA` 寄存器并断言芯片 ID 为 `0x48`，随后触发一次单次测量，并创建高优先级线程 `ak8975_thread`。

线程每隔 `sample_period_ms` 毫秒读取六个数据寄存器，把原始轴映射为 `(-X, +Y, -Z)`，减去校准偏移，乘以各轴缩放，再用 `rotation`（传感器坐标系到应用坐标系）旋转，发布结果并触发下一次单次测量。发布值是传感器原始计数；偏移与缩放的初值为 0 与 1。

`RequestMagCalibration()` 请求硬铁/软铁校准。线程随后在 15 s 内记录各轴最小值与最大值，此期间需把传感器向各方向转动；结束时偏移取中点，缩放取各轴平均量程除以该轴量程。校准结果只保存在 RAM 中。

模块在 RamFS 中注册命令 `ak8975`：

- `ak8975`：打印用法。
- `ak8975 show <time_ms> <interval_ms>`：在 `time_ms` 内每隔 `interval_ms` 打印一次最新磁场向量，`interval_ms` 限制在 10 到 1000 ms。

`OnMonitor()` 在最新数据含 NaN 时输出一条警告日志。

Upon construction, AK8975 configures the SPI (clock polarity high, sampling on the second edge, prescaler DIV_4), reads the `WIA` register and asserts that the chip ID is `0x48`, then triggers one single measurement and creates the high-priority thread `ak8975_thread`.

Every `sample_period_ms` milliseconds the thread reads the six data registers, maps the raw axes to `(-X, +Y, -Z)`, subtracts the calibration offset, multiplies by the per-axis scale, rotates the result with `rotation` (sensor frame to application frame), publishes it and triggers the next single measurement. The published values are raw sensor counts; the offset and scale start at 0 and 1.

`RequestMagCalibration()` requests a hard-iron / soft-iron calibration. The thread then records the per-axis minimum and maximum for 15 s, during which the sensor is rotated in all directions; at the end the offset is the midpoint and the scale is the average axis range divided by each axis range. The calibration result is kept in RAM only.

The Module registers the command `ak8975` in RamFS:

- `ak8975`: print the usage.
- `ak8975 show <time_ms> <interval_ms>`: print the latest magnetic field vector every `interval_ms` for `time_ms`, with `interval_ms` limited to 10 to 1000 ms.

`OnMonitor()` logs a warning when the latest data contains NaN.

## 2. 构造接口 / Constructor

```cpp
AK8975(LibXR::SPI& spi,
       LibXR::RamFS& ramfs,
       LibXR::Quaternion<float>&& rotation = {1.0f, 0.0f, 0.0f, 0.0f},
       const char* data_topic_name = "ak8975_mag",
       uint32_t sample_period_ms = 20,
       size_t task_stack_depth = 1024);
```

依赖：

- `spi`：AK8975 所在的 `LibXR::SPI`，取自 BSP 的硬件注册（`XR_REGISTER`）。
- `ramfs`：接收 `ak8975` 命令的 `LibXR::RamFS`，取自 BSP 的硬件注册。

配置参数：

- `rotation`：从传感器坐标系到应用坐标系的四元数 `{w, x, y, z}`，默认单位四元数。
- `data_topic_name`：发布的 Topic 名称，默认 `"ak8975_mag"`。
- `sample_period_ms`：两次采样之间的休眠时间，单位 ms，默认 20。
- `task_stack_depth`：采样线程的栈深，默认 1024。

Dependencies:

- `spi`: the `LibXR::SPI` of the AK8975, taken from the BSP's Registration (`XR_REGISTER`).
- `ramfs`: the `LibXR::RamFS` that receives the `ak8975` command, taken from the BSP's Registration.

Configuration parameters:

- `rotation`: quaternion `{w, x, y, z}` from the sensor frame to the application frame, default identity.
- `data_topic_name`: name of the published Topic, default `"ak8975_mag"`.
- `sample_period_ms`: sleep between two samples in ms, default 20.
- `task_stack_depth`: stack depth of the sampling thread, default 1024.

## 3. Topic

| Topic | 方向 | 类型 | 说明 |
| --- | --- | --- | --- |
| `data_topic_name`（默认 `ak8975_mag`） | 发布 | `Eigen::Matrix<float, 3, 1>` | 校准并旋转后的磁场向量，单位为原始计数 |

| Topic | Direction | Type | Meaning |
| --- | --- | --- | --- |
| `data_topic_name` (default `ak8975_mag`) | Publish | `Eigen::Matrix<float, 3, 1>` | Calibrated and rotated magnetic field vector in raw counts |

## 4. 配置示例 / Configuration Example

`xrobot instance add xrobot-org/AK8975` 写入的实例，`spi` 与 `ramfs` 填写为 BSP 中注册的名称：

An instance written by `xrobot instance add xrobot-org/AK8975`, with `spi` and `ramfs` set to names registered by the BSP:

```yaml
modules:
  - module: xrobot-org/AK8975
    id: ak8975_0
    args:
      - spi: spi_ak8975
      - ramfs: ramfs
      - rotation: '{1.0f, 0.0f, 0.0f, 0.0f}'
      - data_topic_name: "ak8975_mag"
      - sample_period_ms: 20
      - task_stack_depth: 1024
```

## 5. 依赖与硬件 / Dependencies and Hardware

依赖：LibXR。

硬件：一片通过 SPI 连接的 AK8975。SPI 句柄由 BSP 准备；传感器与其他器件共用同一条物理 SPI 总线时，片选与总线互斥由该句柄负责。

Dependencies: LibXR.

Hardware: one AK8975 connected over SPI. The SPI handle is prepared by the BSP; when the sensor shares a physical SPI bus with other devices, chip-select handling and bus locking are done by that handle.
