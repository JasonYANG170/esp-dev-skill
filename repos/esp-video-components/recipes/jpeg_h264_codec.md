# JPEG / H.264 硬件编解码（M2M 设备）

> **适用摘要**: 使用 esp_video 的硬件编解码设备（JPEG 编码 `/dev/video10`、JPEG 解码 `/dev/video12`、H.264 编码 `/dev/video11`）进行 memory-to-memory（M2M）压缩。典型用于存图、视频流、UVC gadget。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-video-components/resources/`, source/examples in `repos/esp-video-components/`, and this recipe path `repos/esp-video-components/recipes/jpeg_h264_codec.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "JPEG 编码"
- "H.264 编码"
- "硬件编解码"
- "M2M 设备"
- "压缩图像/视频"

## 前置条件

| 设备 | 节点 | Kconfig | 芯片要求 |
|---|---|---|---|
| JPEG 编码 | `/dev/video10` | `ESP_VIDEO_ENABLE_HW_JPEG_ENC_VIDEO_DEVICE=y` | `SOC_JPEG_CODEC_SUPPORTED`（P4/S31） |
| JPEG 解码 | `/dev/video12` | `ESP_VIDEO_ENABLE_HW_JPEG_DEC_VIDEO_DEVICE=y` | `SOC_JPEG_CODEC_SUPPORTED`（P4/S31） |
| H.264 编码 | `/dev/video11` | `ESP_VIDEO_ENABLE_HW_H264_VIDEO_DEVICE=y` | 仅 ESP32-P4（依赖 `esp_h264`） |
| 参考示例 | `esp_video/examples/image_storage/sd_card`、`esp_video/examples/uvc` | | |

## 分步说明

### 1. menuconfig 启用编解码设备

```
Component config  --->
    Espressif Video Configuration  --->
        [*] Enable Hardware JPEG Encoder based Video Device
        [*] Enable Hardware H.264 based Video Device       # 仅 P4
```

### 2. 打开采集设备与编码设备

```c
#include "esp_video_device.h"
#include <fcntl.h>
#include <sys/ioctl.h>
#include "linux/videodev2.h"

int cap_fd = open(ESP_VIDEO_MIPI_CSI_DEVICE_NAME, O_RDONLY);   /* /dev/video0 */
int m2m_fd = open(ESP_VIDEO_JPEG_DEVICE_NAME, O_RDONLY);        /* /dev/video10 */
```

### 3. 设置编码质量 / H.264 码率（ext_ctrls）

```c
struct v4l2_ext_control control[1];
struct v4l2_ext_controls controls = {
    .ctrl_class = V4L2_CID_JPEG_CLASS,
    .count = 1,
    .controls = control,
};
control[0].id = V4L2_CID_JPEG_COMPRESSION_QUALITY;
control[0].value = 80;                       /* 质量 0-100 */
ioctl(m2m_fd, VIDIOC_S_EXT_CTRLS, &controls);
```

H.264 控制项（`V4L2_CID_CODEC_CLASS`）：
- `V4L2_CID_MPEG_VIDEO_H264_I_PERIOD`
- `V4L2_CID_MPEG_VIDEO_BITRATE`
- `V4L2_CID_MPEG_VIDEO_H264_MIN_QP` / `_MAX_QP`（要求 MAX_QP > MIN_QP）

### 4. M2M 双缓冲模型

M2M 设备有两路缓冲，需分别管理：

| 缓冲类型 | 用途 |
|---|---|
| `V4L2_BUF_TYPE_VIDEO_OUTPUT` | 输入：待编码的原始帧（RGB/YUV） |
| `V4L2_BUF_TYPE_VIDEO_CAPTURE` | 输出：编码后的 JPEG/H.264 流 |

```c
/* 简化流程（完整见 image_storage/uvc 示例） */
const int out_type = V4L2_BUF_TYPE_VIDEO_OUTPUT;
const int cap_type = V4L2_BUF_TYPE_VIDEO_CAPTURE;

/* 各自 REQBUFS / QUERYBUF / mmap / QBUF / STREAMON */
/* 循环：
   1. 从摄像头 DQBUF 拿到原始帧
   2. 把原始帧填入 OUTPUT 缓冲并 QBUF
   3. 从 CAPTURE 路 DQBUF 拿到压缩数据，写出/存储
   4. 两路缓冲各自 QBUF 归还 */
ioctl(m2m_fd, VIDIOC_STREAMON, &out_type);
ioctl(m2m_fd, VIDIOC_STREAMON, &cap_type);
```

> 这是 M2M 模型的核心：输入与输出是两套独立的 V4L2 缓冲队列，必须同时启动、同时归还。

### 5. JPEG 解码（`/dev/video12`）

```c
int dec_fd = open(ESP_VIDEO_JPEG_DEC_DEVICE_NAME, O_RDONLY);   /* /dev/video12 */
/* OUTPUT 路：输入 JPEG 数据；CAPTURE 路：输出像素数据
   解码输出格式可用 V4L2_PIX_FMT_BGR565（esp_video_ioctl.h 定义） */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译报 `ESP_VIDEO_JPEG_DEVICE_NAME` 未定义 | 未启用 JPEG 编码 Kconfig | menuconfig 开 `ESP_VIDEO_ENABLE_HW_JPEG_ENC_VIDEO_DEVICE` |
| H.264 在 S3 报错 | H.264 仅 P4 支持 | 换 ESP32-P4 |
| `H264_MAX_QP <= MIN_QP` 编译报错 | Kconfig 校验失败 | 确保 `CONFIG_EXAMPLE_H264_MAX_QP > CONFIG_EXAMPLE_H264_MIN_QP` |
| 编码无输出 | M2M 只管理了一路缓冲 | OUTPUT 与 CAPTURE 两路都要 REQBUFS/QBUF/STREAMON |
| JPEG 质量无变化 | 未用 `V4L2_CID_JPEG_CLASS` | `ctrl_class = V4L2_CID_JPEG_CLASS` |

## 参考项目

- `esp_video/examples/image_storage/sd_card/main/sd_card_main.c` — JPEG/H.264 编码存 SD 卡的完整 M2M 实现
- `esp_video/examples/uvc/main/uvc_example.c` — UVC gadget 中的 JPEG/H.264 编码
- `esp_video/include/esp_video_device.h` — `ESP_VIDEO_JPEG_DEVICE_NAME`、`ESP_VIDEO_H264_DEVICE_NAME`
- `esp_video/Kconfig` — 各编解码设备依赖与 select 关系
