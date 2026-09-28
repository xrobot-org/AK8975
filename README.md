# AK8975

XRobot Module for the AKM AK8975 3-axis magnetometer over SPI.

The constructor configures the SPI handle (CPOL high, CPHA second edge, prescaler
DIV_4), reads the `WIA` register and asserts that the chip ID is `0x48`, then starts
a single measurement and creates the `ak8975_thread` thread (high priority).

Every `sample_period_ms` milliseconds the thread reads the six data registers,
maps the raw axes to `(-X, +Y, -Z)`, subtracts the calibration offset, multiplies by
the per-axis scale, applies `rotation` (sensor frame to application frame),
publishes the result and triggers the next single measurement. Values are raw
sensor counts (no conversion to µT); offset and scale start at 0 and 1.

The SPI handle is prepared by the BSP. If the sensor shares a physical SPI bus with
other devices, chip-select handling and bus locking belong in that handle.

## Topics

| Topic | Type | Content |
| --- | --- | --- |
| `data_topic_name` (default `ak8975_mag`) | `Eigen::Matrix<float, 3, 1>` | Magnetic field vector, calibrated and rotated, raw counts |

## Calibration

`RequestMagCalibration()` requests a hard/soft-iron calibration. The thread then
records the per-axis minimum and maximum for 15 s while the sensor is rotated in all
directions, sets the offset to the midpoint and the scale to the average range divided
by each axis range. The result is kept in RAM only.

## RamFS command

The Module registers the `ak8975` command in RamFS.

- `ak8975`: print usage.
- `ak8975 show <time_ms> <interval_ms>`: print the latest magnetic field vector every
  `interval_ms` (clamped to 10-1000 ms) for `time_ms`.

`OnMonitor()` logs a warning when the latest data contains NaN.

## Dependencies

No other Modules; LibXR only.

## Constructor

```cpp
AK8975(LibXR::SPI& spi,
       LibXR::RamFS& ramfs,
       LibXR::Quaternion<float>&& rotation = {1.0f, 0.0f, 0.0f, 0.0f},
       const char* data_topic_name = "ak8975_mag",
       uint32_t sample_period_ms = 20,
       size_t task_stack_depth = 1024);
```

Dependencies:

- `spi`: `LibXR::SPI` device handle of the AK8975.
- `ramfs`: `LibXR::RamFS` that receives the `ak8975` command.

Configuration:

- `rotation`: quaternion `{w, x, y, z}` from sensor frame to application frame,
  default identity.
- `data_topic_name`: name of the published topic, default `"ak8975_mag"`.
- `sample_period_ms`: sleep between samples in ms, default 20.
- `task_stack_depth`: stack depth of the sampling thread, default 1024.

## Use

```sh
xrobot module add xrobot-org/AK8975
xrobot setup
xrobot instance add xrobot-org/AK8975
```

`xrobot instance add` writes an instance to `User/xrobot.yaml` with empty
dependencies and the source defaults; set the dependencies to the names of objects
the BSP registers with `XR_REGISTER`:

```yaml
modules:
  - module: xrobot-org/AK8975
    id: ak8975_0
    args:
      - spi: spi_ak8975
      - ramfs: ramfs
      - rotation: '{1.0f, 0.0f, 0.0f, 0.0f}'
      - data_topic_name: '"ak8975_mag"'
      - sample_period_ms: '20'
      - task_stack_depth: '1024'
```

BSP side:

```cpp
XR_REGISTER(spi_ak8975, LibXR::SPI);
XR_REGISTER(ramfs, LibXR::RamFS);
```

Run `xrobot setup` again to generate `User/xrobot_main.hpp`.

`xrobot module show .` in this repository, or
`xrobot module show Modules/xrobot-org/AK8975` in a BSP, prints the manifest and the
current constructor.
