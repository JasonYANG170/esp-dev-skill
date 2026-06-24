# USB UVC Gadget（ESP32 作为 USB 摄像头）

> **适用摘要**: 通过 USB 外设把 ESP32 实现成标准 UVC 摄像头设备，主机（PC/手机）免驱识别为 webcam。`uvc` 示例采集摄像头帧 → JPEG/H.264 编码 → UVC 上报。

## 触发意图

- "把 ESP32 做成 USB 摄像头"
- "UVC gadget"
- "USB camera device"
- "webboard / 虚拟摄像头"
- "usb_device_uvc"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | ESP32-P4 / S3（需 USB-OTG） |
| Kconfig | 采集接口 + JPEG 或 H.264 编码设备 |
| 依赖 | `usb_device_uvc`（UVC device 栈） |
| 参考示例 | `esp_video/examples/uvc/main/uvc_example.c` |

## 分步说明

### 1. 初始化视频系统

```c
#include "example_video_common.h"
#include "usb_device_uvc.h"
#include "uvc_frame_config.h"

example_video_init();                  /* 采集 + 编码设备 */
```

### 2. 打开采集与编码设备

```c
#if CONFIG_FORMAT_MJPEG_CAM1
#define ENCODE_DEV_PATH   ESP_VIDEO_JPEG_DEVICE_NAME
#define UVC_OUTPUT_FORMAT V4L2_PIX_FMT_JPEG
#elif CONFIG_FORMAT_H264_CAM1
#define ENCODE_DEV_PATH   ESP_VIDEO_H264_DEVICE_NAME
#define UVC_OUTPUT_FORMAT V4L2_PIX_FMT_H264
#endif

int cap_fd = open(EXAMPLE_CAM_DEV_PATH, O_RDONLY);
int m2m_fd = open(ENCODE_DEV_PATH, O_RDONLY);
```

> H.264 模式要求 `CONFIG_EXAMPLE_H264_MAX_QP > CONFIG_EXAMPLE_H264_MIN_QP`，否则编译报错。

### 3. M2M 编码 → UVC 上报

```c
/* 简化（完整见 uvc_example.c）：
   1. 从 cap_fd DQBUF 原始帧
   2. 填入 m2m_fd 的 OUTPUT 路，QBUF
   3. 从 m2m_fd CAPTURE 路 DQBUF 编码后的 JPEG/H.264
   4. 填入 uvc_fb_t，通过 usb_device_uvc API 上报给主机 */

uvc_fb_t fb;
fb.buf = encoded_buffer;
fb.buf_bytesused = encoded_size;
fb.width = width;
fb.height = height;
fb.format = UVC_OUTPUT_FORMAT;
/* 调用 usb_device_uvc 提供的上报接口（见 usb_device_uvc.h） */
```

> `uvc_example.c` 用 `uvc_t` 结构管理 `cap_fd`、`m2m_fd`、双路缓冲与 `uvc_fb_t`。

### 4. 帧格式配置

UVC 帧格式（分辨率/帧率/格式）通过 `uvc_frame_config.h` 与 menuconfig 的 `CONFIG_FORMAT_MJPEG_CAM1` / `CONFIG_FORMAT_H264_CAM1` 选择，主机枚举时按 UVC 描述符协商。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 主机识别不到 UVC 设备 | USB-OTG 配置 / usb_device_uvc 未启用 | 检查 USB 外设与依赖 |
| 画面卡顿/掉帧 | 编码或 USB 带宽不足 | 降分辨率/质量；JPEG 优于 H.264 |
| H.264 模式编译失败 | MAX_QP <= MIN_QP | 调大 `CONFIG_EXAMPLE_H264_MAX_QP` |
| 主机预览黑屏 | sensor 未稳定 / ISP 未开 | 丢启动帧；RAW 传感器开 ISP Pipeline |
| 非 P4/S3 不可用 | 缺 USB-OTG | 换 P4/S3 |

## 参考项目

- `esp_video/examples/uvc/main/uvc_example.c` — UVC gadget 完整实现
- `esp_video/examples/uvc/main/Kconfig.projbuild` — MJPEG/H264 格式与质量配置
- `esp_video/examples/uvc/main/uvc_frame_config.h` — UVC 帧描述符配置
