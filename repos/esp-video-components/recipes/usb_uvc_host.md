# USB UVC Host（接入 USB 摄像头）

> **适用摘要**: 把 ESP32 作为 USB Host，接入标准 UVC（USB Video Class）USB 摄像头/webcam，通过 `/dev/video40`~`/dev/video49` 采集。需要 USB-OTG（ESP32-P4/S3/S31）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-video-components/resources/`, source/examples in `repos/esp-video-components/`, and this recipe path `repos/esp-video-components/recipes/usb_uvc_host.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "USB 摄像头"
- "webcam"
- "UVC host"
- "ESP32 接 USB 相机"
- "免驱 USB 摄像头"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | ESP32-P4 / S3 / S31（需 `SOC_USB_OTG_SUPPORTED`） |
| Kconfig | `ESP_VIDEO_ENABLE_USB_UVC_VIDEO_DEVICE=y`（默认 n） |
| 依赖 | `usb_host_uvc`（idf_component.yml 中 rules: target in [esp32p4, esp32s3, esp32s31]） |
| 参考 | `esp_video/examples/uvc`（虽是 gadget，但 host 初始化结构同源） |

## 分步说明

### 1. menuconfig 启用 USB-UVC video device

```
Component config  --->
    Espressif Video Configuration  --->
        [*] Enable USB-UVC based Video Device
            (10240) USB UVC Video Device URB Size
            (10000) UVC device initialization timeout (ms)
```

### 2. 填充 usb_uvc 配置

```c
#include "esp_video_init.h"

static const esp_video_init_usb_uvc_config_t usb_uvc_config = {
    .uvc = {
        .uvc_dev_num   = 1,                 /* 预期 USB 摄像头数量 */
        .task_stack    = 4096,
        .task_priority = 5,
        .task_affinity = -1,                /* 无亲和性 */
    },
    .usb = {
        .init_usb_host_lib = true,          /* 让 esp_video 初始化 USB Host 库 */
        .peripheral_map    = 0,              /* 选择 USB 外设 */
        .task_stack    = 4096,
        .task_priority = 5,
        .task_affinity = -1,
    },
};

static const esp_video_init_config_t cam_config = { .usb_uvc = &usb_uvc_config };
esp_video_init(&cam_config);
```

### 3. 打开设备并采集

```c
/* 插入 USB 摄像头后，设备出现在 /dev/video40 起 */
int fd = open(ESP_VIDEO_USB_UVC_DEVICE_NAME(0), O_RDONLY);   /* /dev/video40 */

struct v4l2_capability cap;
ioctl(fd, VIDIOC_QUERYCAP, &cap);

struct v4l2_format format = {
    .type = V4L2_BUF_TYPE_VIDEO_CAPTURE,
    .fmt.pix = { .width = 640, .height = 480,
                 .pixelformat = V4L2_PIX_FMT_MJPEG },   /* UVC 多为 MJPEG */
};
ioctl(fd, VIDIOC_S_FMT, &format);
/* 后续 REQBUFS → mmap → QBUF → STREAMON → DQBUF（MJPEG 需软解或直接传输）
   见 recipes/capture_stream.md */
```

> UVC 设备 ID 范围 40-49，最多 10 个；通过 `ESP_VIDEO_USB_UVC_DEVICE_NAME(n)` 宏取节点名。

### 4. 等待设备枚举

`USB_UVC_INIT_TIMEOUT_MS`（默认 10000ms）控制等待 UVC 设备枚举的超时。热插拔由 USB Host Lib 处理，应用层通过 `VIDIOC_QUERYCAP` 验证设备就绪。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `open("/dev/video40")` 失败 | USB Host 库未初始化或设备未插入 | `init_usb_host_lib=true`；检查供电与连接；增大 `USB_UVC_INIT_TIMEOUT_MS` |
| 非 P4/S3/S31 编译失败 | USB-OTG 不支持 | 换支持的芯片 |
| 设备识别但无 MJPEG | UVC 设备格式不同 | 用 `VIDIOC_ENUM_FMT` 枚举实际格式 |
| 供电不足 | USB 摄像头功耗大 | 外接供电 / 有源 USB hub |
| 多个 UVC 设备 ID 混淆 | 热插拔顺序变化 | 用 `ESP_VIDEO_USB_UVC_DEVICE_NAME(n)` 按序号访问 |

## 参考项目

- `esp_video/idf_component.yml` — `usb_host_uvc` 依赖与 target rules
- `esp_video/include/esp_video_init.h` — `esp_video_init_usb_uvc_config_t`
- `esp_video/include/esp_video_device.h` — `ESP_VIDEO_USB_UVC_DEVICE_NAME` / ID 范围
- `esp-iot-solution/examples/usb/host` — USB 摄像头综合示例（外部仓库）
