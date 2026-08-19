# MIPI-CSI 摄像头采集（ESP32-P4）

> **适用摘要**: 在 ESP32-P4 上配置 MIPI-CSI 接口摄像头（如 SC2336、OV5640、OV2640 等 MIPI 型号），完成初始化与 RAW/YUV/RGB 数据采集。MIPI-CSI 仅 ESP32-P4 支持。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-video-components/resources/`, source/examples in `repos/esp-video-components/`, and this recipe path `repos/esp-video-components/recipes/mipi_csi.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "MIPI-CSI 摄像头"
- "ESP32-P4 摄像头"
- "SC2336 / OV5640 MIPI"
- "RAW 传感��采集"
- "MIPI 多 lane 配置"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | ESP32-P4（唯一支持 MIPI-CSI） |
| Kconfig | `ESP_VIDEO_ENABLE_MIPI_CSI_VIDEO_DEVICE=y`（默认 y，依赖 `SOC_MIPI_CSI_SUPPORTED`） |
| RAW 传感器 | 需额外启用 `ESP_VIDEO_ENABLE_ISP_VIDEO_DEVICE` + `ESP_VIDEO_ENABLE_ISP_PIPELINE_CONTROLLER` |
| 参考示例 | `esp_video/examples/capture_stream`（在 P4 + MIPI 板上运行） |

## 分步说明

### 1. menuconfig 启用 MIPI-CSI 与 ISP Pipeline

```
Component config  --->
    Espressif Video Configuration  --->
        [*] Enable MIPI-CSI based Video Device          # 默认 y
        [*] Enable ISP based Video Device                # MIPI 自动 select ISP
            [*] Enable ISP Pipeline Controller           # RAW 传感器必开
```

### 2. 配置 csi_config（ESP32-P4-Function-EV-Board V1.5）

```c
#include "esp_video_init.h"

static const esp_video_init_csi_config_t csi_config = {
    .sccb_config = {
        .init_sccb = true,
        .i2c_config = { .port = 0, .scl_pin = 8, .sda_pin = 7 },
        .freq = 100000,
    },
    .reset_pin = -1,
    .pwdn_pin  = -1,
};
```

> ESP32-P4-EYE 板需要额外 XCLK（pin 11）与 reset/pwdn 引脚，见 `example_video_common_board.h`。

### 3. 初始化并打开设备

```c
static const esp_video_init_config_t cam_config = { .csi = &csi_config };
esp_video_init(&cam_config);

int fd = open(ESP_VIDEO_MIPI_CSI_DEVICE_NAME, O_RDONLY);   /* /dev/video0 */
```

### 4. 设置 RAW8 格式并采集（典型 SC2336 场景）

```c
struct v4l2_format format = {
    .type = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .fmt.pix = {
        .width = 1600, .height = 1200,
        .pixelformat = V4L2_PIX_FMT_SBGGR8,   /* RAW8 Bayer，ISP 会处理 */
    },
};
ioctl(fd, VIDIOC_S_FMT, &format);
/* 后续 REQBUFS → QUERYBUF → mmap → QBUF → STREAMON → DQBUF/QBUF 循环
   见 recipes/capture_stream.md */
```

> 实际可用的像素格式取决于 sensor 驱动 `query_support_formats`，用 `VIDIOC_ENUM_FMT` 枚举。

### 5. 多 lane 配置说明

MIPI 的 lane 数、HS settle、mipi_clk 等参数由 sensor 驱动通过 `esp_cam_sensor_format_t.mipi_info`（`esp_cam_sensor_mipi_info_t`）提供，应用层无需手填。`esp_cam_sensor_mipi_info_t` 字段：`mipi_clk`、`hs_settle`、`lane_num`、`line_sync_en`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 画面偏紫/偏色/黑 | RAW 传感器未启用 ISP Pipeline | menuconfig 开 `ESP_VIDEO_ENABLE_ISP_PIPELINE_CONTROLLER` |
| 非 P4 芯片编译报 MIPI 相关错误 | MIPI-CSI 仅 P4 支持 | 换 ESP32-P4，或改用 DVP/SPI |
| SCCB 通信失败 | I2C 引脚/频率不对 | 核对板子引脚表，频率 100K-400K |
| 帧率低 | lane 数或 MIPI clk 配置不当 | 确认 sensor 驱动 mipi_info；高分辨率用 2-lane |
| LDO 报错 | LDO 被重复初始化 | 设 `dont_init_ldo = true` 让别处管理 |

## 参考项目

- `esp_video/examples/capture_stream` — 在 ESP32-P4-Function-EV-Board 上跑 MIPI-CSI
- `esp_video/examples/common_components/example_video_common/include/boards/esp32-p4-function-ev-board-v1.5/example_video_common_board.h`
- `esp_cam_sensor/sensors/sc2336/`、`ov5640/` 等 MIPI sensor 驱动
