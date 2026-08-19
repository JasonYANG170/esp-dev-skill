# SPI 接口摄像头

> **适用摘要**: 在 ESP32-C3/C5/C6/C61（仅 SPI 可用）以及 ESP32-P4/S3 上配置 SPI（或 Parallel IO）接口的低分辨率摄像头，含双 SPI 摄像头同时使用。SPI 是唯一全系列支持的接口。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-video-components/resources/`, source/examples in `repos/esp-video-components/`, and this recipe path `repos/esp-video-components/recipes/spi_sensor.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "SPI 摄像头"
- "ESP32-C3/C6 摄像头"
- "GC0308 SPI"
- "双摄像头"
- "低分辨率摄像头"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `ESP_VIDEO_ENABLE_SPI_VIDEO_DEVICE=y`（C3/C5/C6/C61 默认 y；P4/S3/S31 默认 **n**，需手动启用） |
| 双摄像头 | 额外 `ESP_VIDEO_ENABLE_THE_SECOND_SPI_VIDEO_DEVICE=y` |
| 参考示例 | `esp_video/examples/capture_stream`（menuconfig 选 SPI） |

## 分步说明

### 1. menuconfig 启用 SPI video device（P4/S3 必须）

```
Component config  --->
    Espressif Video Configuration  --->
        [*] Enable SPI based Video Device
            [*] Enable The Second SPI Video Device   # 双摄像头才勾
```

### 2. 单 SPI 摄像头配置（ESP32-C6 为例）

```c
#include "esp_cam_ctlr_spi.h"
#include "esp_cam_sensor_xclk.h"
#include "esp_video_init.h"

static const esp_video_init_spi_config_t spi_config = {
    .sccb_config = {
        .init_sccb = true,
        .i2c_config = { .port = 0, .scl_pin = 5, .sda_pin = 4 },
        .freq = 100000,
    },
    .reset_pin = -1,
    .pwdn_pin  = -1,
    .intf      = ESP_CAM_CTLR_SPI_CAM_INTF_SPI,
    .io_mode   = ESP_CAM_CTLR_SPI_CAM_IO_MODE_1BIT,
    .spi_port        = 2,
    .spi_cs_pin      = 1,
    .spi_sclk_pin    = 6,
    .spi_data0_io_pin = 7,
    .spi_data1_io_pin = -1,                  /* 1-bit 不用 */
    .xclk_source = ESP_CAM_SENSOR_XCLK_ESP_CLOCK_ROUTER,
    .xclk_pin    = 0,
    .xclk_freq   = 8000000,
};
```

> C6 引脚号取自 example_video_common 的 Customized 表。`intf` 可选 `ESP_CAM_CTLR_SPI_CAM_INTF_SPI` 或 `ESP_CAM_CTLR_SPI_CAM_INTF_PARLIO`（Parallel IO 才支持 2/4-bit）。

### 3. io_mode 选择

| io_mode | 数据位宽 | intf 要求 |
|---|---|---|
| `ESP_CAM_CTLR_SPI_CAM_IO_MODE_1BIT` | 1-bit | SPI 或 PARLIO |
| `ESP_CAM_CTLR_SPI_CAM_IO_MODE_2BIT` | 2-bit | 仅 PARLIO |
| `ESP_CAM_CTLR_SPI_CAM_IO_MODE_4BIT` | 4-bit | 仅 PARLIO |

> SPI 接口只支持 1-bit。>1-bit 需 `intf = ESP_CAM_CTLR_SPI_CAM_INTF_PARLIO` 并填 data1~data3 引脚。

### 4. 双 SPI 摄像头（P4/S3）

```c
static const esp_video_init_spi_config_t spi_config[2] = {
    {
        .sccb_config = { .init_sccb=true,
            .i2c_config={ .port=0, .scl_pin=8, .sda_pin=7 }, .freq=100000 },
        .intf = ESP_CAM_CTLR_SPI_CAM_INTF_SPI,
        .io_mode = ESP_CAM_CTLR_SPI_CAM_IO_MODE_1BIT,
        .spi_port = 2, .spi_cs_pin = 37, .spi_sclk_pin = 4,
        .spi_data0_io_pin = 21, .spi_data1_io_pin = -1,
        .xclk_source = ESP_CAM_SENSOR_XCLK_ESP_CLOCK_ROUTER,
        .xclk_pin = 20, .xclk_freq = 24000000,
        .reset_pin = -1, .pwdn_pin = -1,
    },
    {
        .sccb_config = { .init_sccb=true,
            .i2c_config={ .port=1, .scl_pin=5, .sda_pin=6 }, .freq=100000 },  /* 不同 I2C port */
        .intf = ESP_CAM_CTLR_SPI_CAM_INTF_SPI,
        .io_mode = ESP_CAM_CTLR_SPI_CAM_IO_MODE_1BIT,
        .spi_port = 1, .spi_cs_pin = 38, .spi_sclk_pin = 22,
        .spi_data0_io_pin = 3, .spi_data1_io_pin = -1,
        .xclk_source = ESP_CAM_SENSOR_XCLK_LEDC,
        .xclk_pin = 23, .xclk_freq = 20000000,
        .reset_pin = -1, .pwdn_pin = -1,
        /* LEDC 模式还需 xclk_ledc_cfg，见 video_init.md */
    },
};

static const esp_video_init_config_t cam_config = { .spi = spi_config };
esp_video_init(&cam_config);

/* 两路设备：/dev/video3 与 /dev/video4 */
int fd0 = open(ESP_VIDEO_SPI_DEVICE_0_NAME, O_RDONLY);
int fd1 = open(ESP_VIDEO_SPI_DEVICE_1_NAME, O_RDONLY);
```

> 两路若用同型号 sensor 且 I2C slave 地址相同，必须用不同 I2C port（见 common README Note 2）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| P4/S3 上 `/dev/video3` 不存在 | SPI video device 默认关闭 | menuconfig 启用 `ESP_VIDEO_ENABLE_SPI_VIDEO_DEVICE` |
| 2-bit/4-bit 不工作 | SPI 接口仅支持 1-bit | 改 `intf = ESP_CAM_CTLR_SPI_CAM_INTF_PARLIO` |
| 双摄像头第二路无数据 | 未启用第二路 / I2C 冲突 | 开 `ESP_VIDEO_ENABLE_THE_SECOND_SPI_VIDEO_DEVICE`；两路用不同 I2C port |
| XCLK 无输出 | 时钟源/引脚错 | C3/C6 用 `ESP_CAM_SENSOR_XCLK_ESP_CLOCK_ROUTER`；按板子选 LEDC 或 clock router |
| 数据错位 | MSB/LSB 顺序 | SPI 仅 1-bit，检查 sensor 寄存器配置 |

## 参考项目

- `esp_video/examples/common_components/example_video_common/README.md` — SPI CAM0/CAM1 各板引脚表 + 双 SPI 配置步骤
- `esp_video/include/esp_video_init.h` — `esp_video_init_spi_config_t` 定义
- `esp_cam_sensor/include/esp_cam_ctlr_spi.h` — `ESP_CAM_CTLR_SPI_CAM_*` 枚举
