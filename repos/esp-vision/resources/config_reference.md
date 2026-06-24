# ESP-VISION 配置参考

> 来源：仓库 `boards/<BOARD>/boardconfig.h`、`boards/<BOARD>/imlib_config.h`、`boards/<BOARD>/board.cmake`、`boards/<BOARD>/port/mpconfigboard.h`、`micropython.cmake`、`Makefile`。以下为真实存在的配置项；不存在则不写。
> API 可用性的权威来源：`micropython.cmake`、`boards/*/port/mpconfigboard.cmake` 与各 `board.cmake`。

## 构建变量（Makefile / idf_ext.py）

| 变量 | 默认 | 说明 |
|---|---|---|
| `BOARD` | `ESP32_P4X_EYE` | 选板，须存在 `boards/<BOARD>/port/mpconfigboard.cmake` |
| `ESPPORT` | — | 串口设备路径，如 `/dev/ttyACM0`（传给 `-p`） |
| `ESPBAUD` | — | 烧录波特率（传给 `-b`） |
| `ESP_IDF_VERSION` | 由 IDF 环境注入 | 决定 `build/micropython/idf<ver>/` 与 IDF overlay 分支 |
| `MP_BASE_REF` | `v1.28.0` | 固定的 MicroPython 基线标签（提交 `e0e9fbb17ed6fd06bb76e266ae554784c9c80804`） |
| `MICROPY_OVERLAY_TARGET` | `build` | `build`=导出到 build 副本；`lib`=检视生成 diff（会动 `lib/micropython`） |

构建约束：
- `ESP32_S31_KORVO` 当前限定 ESP-VISION IDF `master` overlay（`board.cmake` 校验），构建前需 source IDF master 环境。
- `prepare-micropython` 会校验 `lib/micropython` 在固定提交；`lib/micropython` 保持干净。

## 板级配置宏（`boards/<BOARD>/boardconfig.h`）

以 `ESP32_P4X_EYE` 为例（其它板数值不同，宏名一致）：

### 板标识与通用
| 宏 | 示例值 | 说明 |
|---|---|---|
| `ESP_VISION_BOARD_ARCH` | `"ESP32P4"` | 板所属架构字符串 |
| `ESP_VISION_BOARD_TYPE` | `"ESP32_P4X_EYE"` | 板类型字符串 |
| `ESP_VISION_PORT_ESP32` | `1` | ESP32 port 标记 |
| `ESP_VISION_IMLIB_PROFILER_ENABLE` | `0` | imlib 性能分析开关 |
| `ESP_VISION_IMLIB_GPU_ENABLE` | `0` | imlib GPU 开关 |
| `ESP_VISION_IMLIB_JPEG_CODEC_ENABLE` | `0` | imlib 内建 JPEG codec 开关 |
| `ESP_VISION_CACHE_LINE_SIZE` | `32` | cache 行大小 |
| `ESP_VISION_ALLOC_ALIGNMENT` | `32` | 分配对齐（= cache line） |
| `ESP_VISION_DMA_ALIGNMENT` | `32` | DMA 对齐 |

### JPEG 质量
| 宏 | 示例值 | 说明 |
|---|---|---|
| `ESP_VISION_JPEG_QUALITY_LOW` | `60` | 低质量档 |
| `ESP_VISION_JPEG_QUALITY_HIGH` | `60` | 高质量档 |
| `ESP_VISION_JPEG_QUALITY_THRESHOLD` | `320*240*2` | 质量/体积切换阈值（像素字节数） |

### 摄像头
| 宏 | 示例值 | 说明 |
|---|---|---|
| `ESP_VISION_CAMERA_SENSOR_ID` | `0x2710` | sensor ID |
| `ESP_VISION_CAMERA_RAW_INPUT_WIDTH/HEIGHT` | `1280` / `720` | 原始输入尺寸 |
| `ESP_VISION_CAMERA_ACTIVE_INPUT_WIDTH/HEIGHT` | `640` / `480` | 有效输入尺寸 |
| `ESP_VISION_CAMERA_PPA_OUTPUT_QQVGA_WIDTH/HEIGHT` | `160` / `120` | PPA 输出 QQVGA（仅 P4） |
| `ESP_VISION_CAMERA_PPA_OUTPUT_QVGA_WIDTH/HEIGHT` | `320` / `240` | PPA 输出 QVGA（仅 P4） |
| `ESP_VISION_CAMERA_BUFFER_COUNT` | `2` | 帧缓冲数（双缓冲） |
| `ESP_VISION_CAMERA_SCCB_I2C_PORT` | `0` | SCCB（I2C）端口 |
| `ESP_VISION_CAMERA_SCCB_I2C_SCL_PIN` / `_SDA_PIN` | `13` / `14` | SCCB 引脚 |
| `ESP_VISION_CAMERA_SENSOR_RESET_PIN` | `26` | sensor 复位引脚 |
| `ESP_VISION_CAMERA_SENSOR_PWDN_PIN` | `12` | sensor power-down 引脚 |
| `ESP_VISION_CAMERA_XCLK_PIN` | `11` | XCLK 引脚 |
| `ESP_VISION_CAMERA_SCCB_I2C_FREQ` | `100000` | SCCB 频率 |
| `ESP_VISION_CAMERA_XCLK_FREQ` | `24000000` | XCLK 频率 |

### LCD
| 宏 | 示例值 | 说明 |
|---|---|---|
| `ESP_VISION_LCD_WIDTH` / `HEIGHT` | `240` / `240` | 屏幕分辨率 |
| `ESP_VISION_LCD_SPI_HOST` | `2` | LCD 所用 SPI host |
| `ESP_VISION_LCD_PIXEL_CLOCK_HZ` | `80*1000*1000` | 像素时钟 |
| `ESP_VISION_LCD_BPP` | `16` | 每像素位 |
| `ESP_VISION_LCD_PIN_MOSI/CLK/CS/DC/RST/BL` | `16/17/18/19/15/20` | LCD 引脚 |
| `ESP_VISION_LCD_BACKLIGHT_CH/TIMER/PWM_HZ/OUTPUT_INVERT` | `0/0/5000/1` | 背光 PWM 配置 |

### SD 卡
| 宏 | 示例值 | 说明 |
|---|---|---|
| `ESP_VISION_SDCARD_MOUNT_PATH` | `"/sdcard"` | 挂载点 |
| `ESP_VISION_SDCARD_SLOT` | `0` | SDMMC slot |
| `ESP_VISION_SDCARD_BUS_WIDTH` | `4` | 总线宽度 |
| `ESP_VISION_SDCARD_EN_PIN` / `_EN_ACTIVE_LEVEL` | `46` / `0` | 使能引脚与有效电平 |
| `ESP_VISION_SDCARD_DETECT_PIN` / `_DETECT_PRESENT_LEVEL` | `45` / `0` | 检测引脚与在位电平 |
| `ESP_VISION_SDCARD_LDO_CHAN_ID` | `4` | LDO 通道 |

## imlib 算法开关（`boards/<BOARD>/imlib_config.h`）

`IMLIB_ENABLE_*` 决定编译哪些可选算法；未启用的方法调用时抛异常。`OMV_NO_GPL` 必须保留（未经许可证审查不得移除）。

```c
#ifndef OMV_NO_GPL
#define OMV_NO_GPL              // 必须：排除 GPL 条件代码路径
#endif

#define IMLIB_ENABLE_BINARY_OPS
#define IMLIB_ENABLE_MATH_OPS
#define IMLIB_ENABLE_LAB_LUT
#define IMLIB_ENABLE_IMAGE_IO
#define IMLIB_ENABLE_IMAGE_FILE_IO

// 邻域 / 秩 / 保边滤波
#define IMLIB_ENABLE_MEAN
#define IMLIB_ENABLE_MEDIAN
#define IMLIB_ENABLE_MODE
#define IMLIB_ENABLE_MIDPOINT
#define IMLIB_ENABLE_MORPH
#define IMLIB_ENABLE_GAUSSIAN
#define IMLIB_ENABLE_LAPLACIAN
#define IMLIB_ENABLE_BILATERAL

// 几何 / 标记检测
#define IMLIB_ENABLE_FIND_LINES
#define IMLIB_ENABLE_FIND_CIRCLES
#define IMLIB_ENABLE_FIND_RECTS
#define IMLIB_FIND_TEMPLATE
#define IMLIB_ENABLE_QRCODES
#define IMLIB_ENABLE_APRILTAGS
#define IMLIB_ENABLE_APRILTAGS_TAG16H5
#define IMLIB_ENABLE_APRILTAGS_TAG25H7
#define IMLIB_ENABLE_APRILTAGS_TAG25H9
#define IMLIB_ENABLE_APRILTAGS_TAG36H10
#define IMLIB_ENABLE_APRILTAGS_TAG36H11
```

> 来源：`boards/ESP32_P4X_EYE/imlib_config.h`。

## 板级 CMake 开关（`boards/<BOARD>/board.cmake`）

| 变量 | 说明 |
|---|---|
| `ESP_VISION_ENABLE_BARCODE` | `ON`/`OFF`，启用 ZXing-C++ 1D/2D 条码后端（`find_barcodes`）。当前 P4 板设为 `ON`，其它板默认 `OFF`。 |

> 该变量经 `micropython.cmake` 条件链接 `idf::zxing`；仅 ESP32-P4 板级当前启用。

## micropython.cmake — 模块/平台源码与芯片条件

`micropython.cmake` 是集成枢纽。关键点：
- `ESP_VISION_MODULE_SOURCES` — 暴露给 Python 的 C 绑定列表（`py_display.c` / `py_image.c` / `py_imageio.c` / `py_helper.c` / `py_sensor.c`，以及 `py_espdl.cpp`）。
- `h264` 与 `rtsp` 模块仅在 `IDF_TARGET == esp32p4` 时加入。
- 板级存在 `camera.c` / `display.c` / `sdcard.c` 时自动选用，覆盖平台默认。
- 条件链接 `idf::zxing`（取决于板级 `ESP_VISION_ENABLE_BARCODE`）。
- 设置 `OMV_NO_GPL=1`；拉入 `ulab`。
- `platform/main.c` 作为 MicroPython 主源注入，启动初始化 + 软复位循环。

> 移除/新增模块应同步处理 `target_sources`、`target_link_libraries`、组件清单与 size 报告。

## 标准 MicroPython 功能宏（`boards/<BOARD>/port/mpconfigboard.h`）

`MICROPY_PY_*` / `MICROPY_HW_*` 覆盖单板的标准 MicroPython 功能。当前所有板配置均关闭 Bluetooth 与 ESP-NOW；S31 板还关闭 `machine.ADC` / `machine.ADCBlock`。`MICROPY_PY_NETWORK_WLAN` 取决于网络硬件与固件配置。

```c
#define MICROPY_PY_BLUETOOTH       (0)
#define MICROPY_PY_ESPNOW          (0)
#define MICROPY_PY_NETWORK_WLAN    (1)
```

> 这些宏可能依赖 ESP-IDF 版本与 SoC 能力宏；关闭后可能还需移除对应 ESP-IDF 配置或托管依赖才能获得可观测体积缩减。

## 冻结 Python（`boards/<BOARD>/manifest.py`）

```python
freeze("$(PORT_DIR)/modules")
freeze("$(ESP_VISION_ROOT)/modules", "py_inisetup.py")
freeze("$(ESP_VISION_ROOT)/boards/<BOARD>", "board_inisetup.py")
include("$(MPY_DIR)/extmod/asyncio")    # asyncio 已冻结进固件
```

可用 `freeze()` / `module()` / `package()` / `include()` 增减冻结代码；不需要时删对应条目。

## 分区表与启动文件

- 分区表：`boards/<BOARD>/port/partitions-*.csv`（如 `partitions-16MiB-esp-vision.csv`、`partitions-8MiB-esp-vision.csv`）。
- 板级 sdkconfig 链：`sdkconfig.board` + 板专用 `sdkconfig.<short>`（如 `sdkconfig.p4x_eye`、`sdkconfig.s3_eye`、`sdkconfig.s31_korvo`）。
- 启动文件：冻结的 `_boot.py`（挂载内部 Flash 到 `/`）→ `py_inisetup.py`（初始化/修复文件系统）→ `/boot.py`（可选）→ `/main.py`（产品入口）。板级默认 `main.py`/`README.txt` 内容可由 `boards/<BOARD>/board_inisetup.py` 提供。
