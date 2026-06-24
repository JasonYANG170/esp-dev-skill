# 摄像头选型与初始化

> **适用摘要**: 根据 SoC 选择正确的 `WhoCam` 子类（S3 的 `WhoS3Cam` / P4 的 `WhoP4Cam` / USB 的 `WhoUVCCam`），正确传像素格式、帧尺寸、翻转方向与 fb_count。

## 触发意图

- "选摄像头"
- "WhoS3Cam 参数"
- "WhoP4Cam 怎么初始化"
- "USB UVC 摄像头"
- "帧尺寸 framesize"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `who_cam.hpp`（按 target 自动 include 对应子类） |
| 组件 | `who_cam/idf_component.yml`：S3 拉 esp32-camera；P4 拉 esp_video；都拉 esp-dl 与 usb_host_uvc |

## 分步说明

### 1. 头文件按 target 自动切换

`who_cam.hpp` 内容（来自仓库）：
```cpp
#include "sdkconfig.h"
#if CONFIG_IDF_TARGET_ESP32S3
#include "who_s3_cam.hpp"
#elif CONFIG_IDF_TARGET_ESP32P4
#include "who_p4_cam.hpp"
#endif
#include "who_uvc_cam.hpp"
```

只需 `#include "who_cam.hpp"`。

### 2. `cam_fb_t` 帧结构（来自 `who_cam_define.hpp`）

```cpp
typedef struct cam_fb_s {
    void *buf; size_t len;
    uint16_t width, height;
    cam_fb_fmt_t format;          // CAM_FB_FMT_RGB565 / RGB888 / JPEG / UKN
    struct timeval timestamp;
    void *ret;
    operator dl::image::img_t() const;   // 隐式转 ESP-DL 图像
} cam_fb_t;
```

### 3. ESP32-S3：`WhoS3Cam`（基于 esp32-camera）

构造：`WhoS3Cam(pixel_format, frame_size, fb_count, vertical_flip=false, horizontal_flip=true)`。

```cpp
#include "who_cam.hpp"
using namespace who::cam;

// 用 BSP LCD 分辨率自动选最大不超过的 framesize
framesize_t fs = get_cam_frame_size_from_lcd_resolution();
auto cam = new WhoS3Cam(PIXFORMAT_RGB565, fs, 5);

// Korvo-2 需要双向翻转
#ifdef BSP_BOARD_ESP32_S3_KORVO_2
auto cam = new WhoS3Cam(PIXFORMAT_RGB565, fs, 5, true, true);
#endif

// 二维码示例固定用 240x240
auto cam = new WhoS3Cam(PIXFORMAT_RGB565, FRAMESIZE_240X240, 4);
```

| 参数 | 类型 | 说明 |
|---|---|---|
| `pixel_format` | `pixformat_t` | `PIXFORMAT_RGB565`（首选，喂 ESP-DL）/ `PIXFORMAT_RGB888` / `PIXFORMAT_JPEG` |
| `frame_size` | `framesize_t` | esp32-camera 枚举，如 `FRAMESIZE_240X240`；可用 `get_cam_frame_size_from_lcd_resolution()` 自动选 |
| `fb_count` | `uint8_t` | 至少 4；LCD 模式 `MODEL_TIME+3` |
| `vertical_flip` | bool | 默认 false |
| `horizontal_flip` | bool | 默认 true（水平翻转） |

取/还帧：`cam->cam_fb_get()` / `cam->cam_fb_return(fb)`（一般不直接调，由 `WhoFetchNode` 代劳）。

### 4. ESP32-P4：`WhoP4Cam`（基于 V4L2 / esp_video）

构造：`WhoP4Cam(v4l2_fmt, fb_count, fb_mem_type=V4L2_MEMORY_USERPTR, vertical_flip=false, horizontal_flip=true)`。

```cpp
#include "who_cam.hpp"
using namespace who::cam;

auto cam = new WhoP4Cam(V4L2_PIX_FMT_RGB565, 5);
// 其它格式：V4L2_PIX_FMT_RGB24 (→RGB888)、V4L2_PIX_FMT_MJPEG / V4L2_PIX_FMT_JPEG
```

P4 默认配 SC2336 sensor，对应 sdkconfig（来自 `sdkconfig.bsp.esp32_p4_function_ev_board`）：
```
CONFIG_CAMERA_SC2336=y
CONFIG_CAMERA_SC2336_MIPI_RAW8_1024X600_30FPS=y
CONFIG_CAMERA_SC2336_CUSTOMIZED_IPA_JSON_CONFIGURATION_FILE=y
CONFIG_CAMERA_SC2336_CUSTOMIZED_IPA_JSON_CONFIGURATION_FILE_PATH="../../components/who_peripherals/who_cam/who_p4_cam/sc2336.json"
```

### 5. USB UVC：`WhoUVCCam`

构造：`WhoUVCCam(fmt, h_res, v_res, fps, fb_count)`。输出通常为 MJPEG，需下游 `WhoDecodeNode`。

```cpp
#include "who_cam.hpp"
using namespace who::cam;

auto cam = new WhoUVCCam(UVC_VS_FORMAT_MJPEG, 640, 480, 30, 4);
// 之后流水线必须加 WhoDecodeNode 解码为 RGB565
```

UVC 依赖 USB Host：`WhoUSB` 单例任务。P4 的 sdkconfig 需要：
```
CONFIG_USB_HOST_CONTROL_TRANSFER_MAX_SIZE=4096
CONFIG_USB_HOST_HW_BUFFER_BIAS_IN=y
CONFIG_USB_HOST_HUBS_SUPPORTED=y
```

### 6. 选型决策表

| 场景 | 选 | 说明 |
|---|---|---|
| ESP32-S3-EYE / Korvo-2 板载 DVP camera | `WhoS3Cam` | 用 `get_cam_frame_size_from_lcd_resolution()` 选 framesize |
| ESP32-P4 Function EV Board（MIPI SC2336） | `WhoP4Cam` | `V4L2_PIX_FMT_RGB565` |
| 任意外接 USB 摄像头 | `WhoUVCCam` | 需 Decode 节点；P4 可再接 PPA 缩放 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `WhoS3Cam` 在 P4 编译失败 | target 搞错 | P4 用 `WhoP4Cam`；S3 用 `WhoS3Cam`（由 `who_cam.hpp` 自动切换） |
| Korvo-2 画面倒置 | 没开双向翻转 | `new WhoS3Cam(..., MODEL_TIME+3, true, true)` |
| `cam_fb_get` 返回 UKN 格式 | 像素格式不支持 | RGB565/RGB888/JPEG 之外的格式被归为 UKN，下游模型无法处理 |
| UVC 无图像 | USB Host 未起 / 缺 Decode | 确认 `WhoUSB` 单例被 App 间接启动；流水线加 `WhoDecodeNode` |
| framesize 超过 LCD 分辨率 | `get_cam_frame_size_from_lcd_resolution()` 已自动取最大不超过的 | 不要手动塞超过 `BSP_LCD_H/V_RES` 的 framesize |

## 参考

- `components/who_peripherals/who_cam/who_cam.hpp`、`who_cam_base.hpp`、`who_cam_define.hpp`
- `components/who_peripherals/who_cam/who_s3_cam/who_s3_cam.hpp`
- `components/who_peripherals/who_cam/who_p4_cam/who_p4_cam.hpp`
- `components/who_peripherals/who_cam/who_uvc_cam/who_uvc_cam.hpp`
- `components/who_peripherals/who_cam/who_p4_cam/sc2336.json`
- `resources/api_reference.md` §4 摄像头
