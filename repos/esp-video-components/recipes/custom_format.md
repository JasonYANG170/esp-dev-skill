# 自定义传感器格式与寄存器序列

> **适用摘要**: 当内置的 sensor 默认格式不满足需求时，通过自定义寄存器初始化序列 + `esp_cam_sensor_format_t` 描述 + `VIDIOC_S_SENSOR_FMT` 命令，让 sensor 按非内置分辨率/格式/帧率工作。

## 触发意图

- "自定义分辨率"
- "传感器寄存器序列"
- "VIDIOC_S_SENSOR_FMT"
- "非标准帧率"
- "自定义 camera format"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `esp_video/examples/video_custom_format/main/` |
| 头文件 | `esp_video_ioctl.h`（`VIDIOC_S_SENSOR_FMT`）、`esp_cam_sensor_types.h`（`esp_cam_sensor_format_t`） |

## 分步说明

### 1. 提供自定义寄存器初始化序列

```c
/* 以 SC2336 为例：800x800 RAW8 30fps，2-lane MIPI，24M 输入 */
const sc2336_reginfo_t app_sc2336_mipi_2lane_24Minput_800x800_raw8_30fps[] = {
    {0x0103, 0x01},
    {0x0100, 0x00},   /* sleep en */
    /* ... 完整寄存器列表来自 sensor datasheet / 厂商配置 ... */
};
```

> 寄存器结构体类型与值必须来自 sensor 驱动头（如 `sc2336_reginfo_t`），不可臆造。

### 2. 提供格式描述信息

```c
#include "esp_cam_sensor_types.h"

const esp_cam_sensor_format_t custom_format_info = {
    .name    = "MIPI_2lane_24Minput_RAW8_800x800_30fps",
    .format  = ESP_CAM_SENSOR_PIXFORMAT_RAW8,
    .port    = ESP_CAM_SENSOR_MIPI_CSI,
    .xclk    = 24000000,
    .width   = 800,
    .height  = 800,
    .regs    = app_sc2336_mipi_2lane_24Minput_800x800_raw8_30fps,
    .regs_size = sizeof(app_sc2336_mipi_2lane_24Minput_800x800_raw8_30fps),
    .fps     = 30,
    /* .isp_info 按需，RAW 传感器需要 */
    .mipi_info = {
        .mipi_clk = 750000000,
        .hs_settle = 10,
        .lane_num = 2,
        .line_sync_en = false,
    },
};
```

`esp_cam_sensor_format_t` 关键字段（见 `esp_cam_sensor_types.h`）：
- `format`：`esp_cam_sensor_output_format_t`（如 `ESP_CAM_SENSOR_PIXFORMAT_RAW8`、`_YUV422`、`_RGB565`、`_JPEG`）
- `port`：`ESP_CAM_SENSOR_DVP` / `_MIPI_CSI` / `_SPI`
- `regs` + `regs_size`：寄存器序列缓冲与长度
- `mipi_info`（MIPI）或 `spi_info`（SPI）：底层 RX 初始化参数

### 3. 用 VIDIOC_S_SENSOR_FMT 应用自定义格式

```c
#include "esp_video_ioctl.h"
#include "esp_video_device.h"
#include <fcntl.h>
#include <sys/ioctl.h>

int fd = open(ESP_VIDEO_MIPI_CSI_DEVICE_NAME, O_RDONLY);

/* 注意：这是私有的 S_SENSOR_FMT，不是标准 VIDIOC_S_FMT */
if (ioctl(fd, VIDIOC_S_SENSOR_FMT, &custom_format_info) != 0) {
    ESP_LOGE(TAG, "failed to set custom sensor format");
}

/* 之后再用标准 VIDIOC_S_FMT 让 video device 适配 sensor 输出 */
struct v4l2_format format = {
    .type = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .fmt.pix = { .width = 800, .height = 800,
                 .pixelformat = V4L2_PIX_FMT_SBGGR8 },
};
ioctl(fd, VIDIOC_S_FMT, &format);
```

> `VIDIOC_S_SENSOR_FMT` / `VIDIOC_G_SENSOR_FMT` 是 esp_video 私有 ioctl（`BASE_VIDIOC_PRIVATE + 1/2`），定义在 `esp_video_ioctl.h`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `VIDIOC_S_SENSOR_FMT` 返回错误 | 寄存器序列或字段不全 | 核对 `regs`/`regs_size`/`port`/`format` 是否齐备 |
| 应用后无图像 | mipi_info/spi_info 与硬件不符 | 核对 lane 数、mipi_clk、hs_settle |
| 误用 `VIDIOC_S_FMT` | 自定义寄存器需用私有 ioctl | 用 `VIDIOC_S_SENSOR_FMT`（来自 `esp_video_ioctl.h`） |
| RAW 格式偏色 | 未启用 ISP Pipeline | 见 `recipes/isp_pipeline.md` |

## 参考项目

- `esp_video/examples/video_custom_format/main/` — 完整自定义格式示例
- `esp_video/examples/video_custom_format/README.md` — 三步法说明
- `esp_video/include/esp_video_ioctl.h` — `VIDIOC_S_SENSOR_FMT` 定义
- `esp_cam_sensor/include/esp_cam_sensor_types.h` — `esp_cam_sensor_format_t`
