# DVP 并行接口摄像头

> **适用摘要**: 在 ESP32-P4 / ESP32-S3 上配置 DVP（Digital Video Port）并行接口摄像头（如 OV2640、GC0308、BF3901 等），完成引脚、XCLK、数据宽度配置与采集。DVP 需要 `SOC_LCDCAM_CAM_SUPPORTED`。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-video-components/resources/`, source/examples in `repos/esp-video-components/`, and this recipe path `repos/esp-video-components/recipes/dvp_sensor.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "DVP 摄像头"
- "并行接口摄像头"
- "OV2640 DVP"
- "ESP32-S3 摄像头"
- "8-bit 并行摄像头"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | ESP32-P4 或 ESP32-S3 |
| Kconfig | `ESP_VIDEO_ENABLE_DVP_VIDEO_DEVICE=y`（默认 y，依赖 `SOC_LCDCAM_CAM_SUPPORTED`） |
| 引脚 | 8 数据线 + VSYNC + DE + PCLK + XCLK（共 12 个 GPIO） |
| 参考示例 | `esp_video/examples/capture_stream`（menuconfig 选 DVP） |

## 分步说明

### 1. menuconfig 选择 DVP 接口

```
Example Video Initialization Configuration  --->
    Select Target Development Board (ESP32-P4-Function-EV-Board V1.5)
    Select and Set Camera Sensor Interface  --->
        [ ] MIPI-CSI
        [*] DVP
```

### 2. 填充 dvp_config（V1.5 引脚）

```c
#include "esp_cam_ctlr_dvp.h"
#include "esp_video_init.h"

static const esp_video_init_dvp_config_t dvp_config = {
    .sccb_config = {
        .init_sccb = true,
        .i2c_config = { .port = 0, .scl_pin = 8, .sda_pin = 7 },
        .freq = 100000,
    },
    .reset_pin = -1,
    .pwdn_pin  = -1,
    .dvp_pin = {
        .data_width = CAM_CTLR_DATA_WIDTH_8,
        .data_io = { 2, 32, 33, 23, 3, 6, 5, 21 },   /* D0..D7 (V1.5) */
        .vsync_io = 37,
        .de_io     = 22,
        .pclk_io   = 4,
        .xclk_io   = 20,
    },
    .xclk_freq = 20000000,                           /* 20 MHz */
};

static const esp_video_init_config_t cam_config = { .dvp = &dvp_config };
esp_video_init(&cam_config);
```

### 3. 打开设备并采集

```c
int fd = open(ESP_VIDEO_DVP_DEVICE_NAME, O_RDONLY);   /* /dev/video2 */

struct v4l2_format format = {
    .type = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .fmt.pix = { .width = 640, .height = 480,
                 .pixelformat = V4L2_PIX_FMT_RGB565 },
};
ioctl(fd, VIDIOC_S_FMT, &format);
/* REQBUFS → QUERYBUF → mmap → QBUF → STREAMON → DQBUF/QBUF
   见 recipes/capture_stream.md */
```

> ESP32-P4 的 DVP 也可经 ISP（`/dev/video1`）处理 RAW，对应 `ESP_VIDEO_ISP_DVP_DEVICE_NAME`。

### 4. 各开发板 DVP 引脚速查（节选自 example_video_common README）

| 信号 | P4-Func-EV V1.5 | ESP32-S3-EYE | ESP32-S31-Korvo |
|---|---|---|---|
| XCLK | 20 | 15 | 55 |
| PCLK | 4 | 13 | 54 |
| VSYNC | 37 | 6 | 56 |
| DE | 22 | 7 | 57 |
| D0..D7 | 2,32,33,23,3,6,5,21 | 11,9,8,10,12,18,17,16 | 46..53 |

> P4-Func-EV V1.4 与 P4-EYE 默认不支持 DVP，需选 `Customized Development Board` 自填。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 画面花屏/错位 | 数据线顺序接错 | 严格按 D0..D7 顺序填 `data_io` 数组 |
| 无图像 | XCLK 未输出 | 确认 `xclk_freq` 与 `xclk_io` 正确，sensor 需要外部时钟 |
| 同步丢失 | VSYNC/DE 引脚错 | 核对 `vsync_io`/`de_io` |
| SCCB 失败 | I2C 引脚/端口不对 | DVP 的 SCCB 与 MIPI 可共用，核对引脚表 |
| SP0A39 传感器不工作 | SP0A39 只能 Parallel IO 驱动 | 仅 P4/C5 支持，SPI slave 模式不行 |

## 参考项目

- `esp_video/examples/common_components/example_video_common/README.md` — 全板 DVP 引脚表
- `esp_video/examples/capture_stream` — DVP 采集示例
- `esp_cam_sensor/sensors/ov2640/`、`gc0308/`、`bf3901/` — DVP sensor 驱动
