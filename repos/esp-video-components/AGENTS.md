# AGENTS.md — Supplementary Agent Guide

> 核心规则、芯片矩阵、设备节点表、陷阱清单、recipes 索引与执行工作流全部在 `SKILL.md`。
> 本文件仅记录 `SKILL.md` 未覆盖的工程约定与工具链指引，不重复内容。

## Project Context

**Language**: C · **Target**: ESP32-P4 / S3 / S31 / C3 / C5 / C6 / C61 · **Framework**: ESP-IDF >= 5.4
**Core components**: `esp_video`（ESPRESSIF MIT）/ `esp_cam_sensor`（Apache-2.0）/ `esp_sccb_intf`（Apache-2.0）/ `esp_ipa`（ESPRESSIF MIT）
**编程模型**: 应用层通过 POSIX（`open`/`ioctl`/`mmap`/`close`）访问 `/dev/videoN`，与 Linux V4L2 兼容；底层驱动通过 `esp_video_init()` 一次性注册。

## Code Generation Conventions

### File Naming
- 应用主文件：`app_main.c`（ESP-IDF 约定，入口 `app_main(void)`）
- 示例主文件：`<scenario>_main.c` 或 `<scenario>_example.c`（如 `capture_stream_main.c`、`uvc_example.c`）
- 板级头文件：`example_video_common_board.h`（按板子放在 `include/boards/<board>/` 下）

### Include 模式（应用层）
```c
#include <fcntl.h>
#include <sys/ioctl.h>
#include <sys/mman.h>
#include "esp_err.h"
#include "esp_log.h"
#include "linux/videodev2.h"        // V4L2 结构与 ioctl 命令
#include "esp_video_device.h"       // 设备节点宏 ESP_VIDEO_*_DEVICE_NAME
#include "esp_video_init.h"         // esp_video_init_config_t 等
#include "esp_video_ioctl.h"        // VIDIOC_S_SENSOR_FMT, V4L2_CTRL_CLASS_ESP_CAM_IOCTL
#include "esp_cam_sensor_types.h"   // esp_cam_sensor_format_t, ESP_CAM_SENSOR_IOC_*
```

> 这些头由 `esp_video` 组件提供；`idf.py add-dependency esp_video` 后即可包含。`linux/videodev2.h` 是仓库自带的 V4L2 头（位于 `esp_video/include/linux/`），非系统 glibc 头。

### 标准工程结构（基于官方示例）
```
my_video_project/
├── CMakeLists.txt
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml          # 依赖 esp_video（可 override_path 指向本地）
│   ├── Kconfig.projbuild          # 示例级配置（格式、引脚、抓拍时长等）
│   └── <scenario>_main.c          # app_main + V4L2 采集循环
├── managed_components/            # esp_video 等托管组件（构建时拉取）
└── sdkconfig                      # ESP_VIDEO_ENABLE_* 选项在此
```

### 规范入口/初始化模式
```c
void app_main(void)
{
    /* 1. 板级/系统初始化（引脚、PSRAM、网络等，按需） */

    /* 2. 初始化 esp_video 子系统（注册 /dev/videoN） */
    ESP_ERROR_CHECK(example_video_init());
    /* example_video_init 内部：esp_video_init(&s_cam_config)
       s_cam_config 按 menuconfig 选择的接口填充 csi/dvp/spi/usb_uvc */

    /* 3. V4L2 采集循环（open + S_FMT + REQBUFS + QUERYBUF + mmap +
          QBUF + STREAMON + DQBUF/QBUF + STREAMOFF + close） */

    /* 4. 反初始化 */
    ESP_ERROR_CHECK(example_video_deinit());
}
```

> `example_video_init()` / `example_video_deinit()` 来自 `common_components/example_video_common`，封装了 `esp_video_init()` 与可选的 XCLK 分配、I2C 总线创建。直接用原生 API 时替换为 `esp_video_init(&cfg)` / `esp_video_deinit()`。

### 日志约定
```c
static const char *TAG = "example";
ESP_LOGI(TAG, "driver: %s", capability.driver);
```
调试默认通过 ESP-IDF monitor（UART）输出，波特率由 `menuconfig` 配置。

## Build Workflow

1. 设置目标芯片：`idf.py set-target esp32p4`（或 esp32s3 / esp32c6 等）
2. 配置：`idf.py menuconfig`
   - `Component config -> Espressif Video Configuration` 启用对应 video device（MIPI/DVP/SPI/USB/ISP/Codec）
   - `Example Video Initialization Configuration` 选板子或 `Customized Development Board` 填引脚
3. 编译：`idf.py build`
4. 烧录监控：`idf.py -p PORT flash monitor`
5. 如需修改组件源码：在 `main/idf_component.yml` 用 `override_path` 指向本地 clone 的 esp_video 目录

## esp_video 应用代码生成 Checklist

- [ ] `esp_video_init(&cfg)` 在任何 `open("/dev/videoN")` 之前调用
- [ ] `esp_video_init_config_t` 仅填充被启用接口对应的字段（csi/dvp/spi/usb_uvc/jpeg_enc/jpeg_dec/cam_motor）
- [ ] SCCB 配置二选一：`init_sccb=true`+`i2c_config` 或 `init_sccb=false`+`i2c_handle`，不混用
- [ ] `reset_pin`/`pwdn_pin` 无硬件引脚时填 `-1`
- [ ] 设备路径用宏（`ESP_VIDEO_MIPI_CSI_DEVICE_NAME` 等），不硬编码字符串
- [ ] 采集流程顺序：S_FMT → REQBUFS → QUERYBUF → mmap → QBUF → STREAMON → DQBUF → QBUF → STREAMOFF
- [ ] DQBUF 后无条件 QBUF 归还缓冲（出错帧也要归还）
- [ ] RAW 传感器已启用 `ESP_VIDEO_ENABLE_ISP_PIPELINE_CONTROLLER`
- [ ] M2M 编解码（JPEG/H.264）同时管理 OUTPUT 与 CAPTURE 两路缓冲
- [ ] SPI video device 在 P4/S3 上需 menuconfig 显式启用
- [ ] sensor ext_ctrls 的 `ctrl_class` 正确：USER（VFLIP/HFLIP）vs ESP_CAM_IOCTL（chip id/reg/gain）
- [ ] USERPTR 模式缓冲用 `heap_caps_aligned_alloc(64, len, MALLOC_CAP_SPIRAM|MALLOC_CAP_CACHE_ALIGNED)`

## Do Not Modify

- `esp_video/`、`esp_cam_sensor/`、`esp_sccb_intf/`、`esp_ipa/` 组件源码 —— 视为只读库，通过 menuconfig 与 `esp_video_init_config_t` 配置
- `SKILL.md` frontmatter（Skill 元数据）
- 修改组件源码需在 `main/idf_component.yml` 用 `override_path` 指向本地副本，不改原始仓库
