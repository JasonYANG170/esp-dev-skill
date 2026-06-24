---
name: esp-video-components-skill
description: >-
  AI Skill for developing camera/video firmware with Espressif esp-video-components.
  Used when users need to create, configure, or debug ESP32 video applications, including
  MIPI-CSI / DVP / SPI / USB-UVC camera capture, V4L2 streaming, JPEG/H.264 codec pipelines,
  image storage, USB-UVC gadget, and ISP pipeline control. Grounded entirely in the real
  repository APIs (esp_video, esp_cam_sensor, esp_sccb_intf, esp_ipa).
  Trigger words: "esp-video-components", "esp_video", "V4L2", "MIPI-CSI", "DVP", "USB-UVC",
  "camera", "摄像头", "视频", "ESP32-P4", "SC2336", "OV5640", "图像采集", "视频流", "JPEG编码", "ISP"
tags:
  - embedded
  - esp-idf
  - espressif
  - camera
  - video
  - V4L2
  - MIPI-CSI
  - DVP
  - ESP32-P4
  - firmware
license: Apache-2.0 (esp_cam_sensor/esp_sccb_intf) ; ESPRESSIF MIT (esp_video/esp_ipa)
compatibility: ESP-IDF >= 5.4 ; targets ESP32-P4, ESP32-S3, ESP32-S31, ESP32-C3, ESP32-C5, ESP32-C6, ESP32-C61
metadata:
  author: Community
  version: "1.0.0"
---

# esp-video-components-skill

面向 Espressif **esp-video-components** 仓库的 AI 开发技能。esp-video-components 是乐鑫官方的摄像头系统开发框架，由 `esp_video`、`esp_cam_sensor`、`esp_sccb_intf`、`esp_ipa` 四个子组件构成，向应用层提供与 Linux V4L2 标准兼容的 POSIX（open/ioctl/mmap）API，可同时管理多路摄像头与多种相机接口（MIPI-CSI、DVP、SPI、USB-UVC）。本技能以仓库真实文档、头文件与示例代码为唯一依据，提供场景化的 recipes、API/配置速查与陷阱清单。

## Core Principles

1. **绝不臆造 API** — esp_video 的对外接口是 POSIX（`open`/`ioctl`/`mmap`/`close`），不是某套 `esp_xxx_capture()`。所有设备节点为 `/dev/videoN`（见 `resources/api_reference.md`），找不到就视为不存在。
2. **初始化顺序固定** — 必须先 `esp_video_init(&config)`（内部完成 SCCB/I2C、LDO、CSI/DVP/SPI/USB/ISP/Codec 子设备注册），再用 POSIX 打开 `/dev/videoN`，最后 ioctl 设格式 → 申请缓冲 → STREAMON → DQBUF/QBUF 循环。
3. **V4L2 缓冲模型是核心** — 采集流程严格遵循 `VIDIOC_S_FMT → VIDIOC_REQBUFS → VIDIOC_QUERYBUF → mmap → VIDIOC_QBUF → VIDIOC_STREAMON → VIDIOC_DQBUF/QBUF → VIDIOC_STREAMOFF`。缓冲类型为 `V4L2_BUF_TYPE_VIDEO_CAPTURE`，内存模式默认 `V4L2_MEMORY_MMAP`（也可 `V4L2_MEMORY_USERPTR`）。
4. **SCCB（I2C）配置二选一** — `esp_video_init_sccb_config_t.init_sccb=true` 时由 `esp_video_init` 内部初始化 I2C；若应用已自行 `i2c_new_master_bus`，则置 `init_sccb=false` 并通过 `i2c_handle` 传入，避免重复初始化冲突。
5. **设备节点编号即接口** — MIPI-CSI=`/dev/video0`、DVP=`/dev/video2`、SPI=`/dev/video3`（第二路 `/dev/video4`）、ISP=`/dev/video20`、JPEG 编码=`/dev/video10`、JPEG 解码=`/dev/video12`、H.264=`/dev/video11`、USB-UVC=`/dev/video40`~`/dev/video49`。
6. **RAW 传感器必须启用 ISP Pipeline** — 输出 RAW8/RAW10/RAW12 的传感器（如 SC2336、OV5640 RAW 模式）需要打开 ISP（`ESP_VIDEO_INIT_FLAGS_ISP`）并启用 `ESP_VIDEO_ENABLE_ISP_PIPELINE_CONTROLLER`，否则无 AE/AWB/颜色还原。
7. **Codec 是 M2M（memory-to-memory）设备** — JPEG/H.264 编解码通过独立的 `/dev/video10|11|12`，用 `V4L2_BUF_TYPE_VIDEO_OUTPUT`（输入原始帧）+ `V4L2_BUF_TYPE_VIDEO_CAPTURE`（输出压缩流）两路缓冲，典型见 `image_storage` 与 `uvc` 示例。
8. **芯片能力决定可用接口** — MIPI-CSI 仅 ESP32-P4；DVP 仅 ESP32-P4/S3；SPI 全系列（含 C3/C5/C6/C61）；USB-UVC 需 USB-OTG（P4/S3/S31）。下表为权威依据。
9. **Sensor 控制走 V4L2 ext_ctrls** — 翻转用 `V4L2_CID_VFLIP/HFLIP`（`V4L2_CTRL_CLASS_USER`）；读芯片 ID、寄存器、增益等走 `V4L2_CTRL_CLASS_ESP_CAM_IOCTL` + `ESP_CAM_SENSOR_IOC_*`（见 `resources/api_reference.md`）。
10. **`reset_pin`/`pwdn_pin` 无则填 -1** — 硬件无复位/掉电引脚时，`gpio_num_t` 字段必须置 `-1`，否则会误配 GPIO。
11. **示例复用优于从零写** — 框架内置 `capture_stream`、`image_storage`、`simple_video_server`、`uvc`、`v4l2_cmd`、`video_custom_format` 六个官方示例；新建项目优先 `idf.py add-dependency esp_video` 再就近改示例。
12. **`example_video_common` 处理板级差异** — 各示例通过该公共组件屏蔽 MIPI/DVP/SPI/USB 的引脚与 SCCB 配置；板子未列出时选 `Customized Development Board` 自填引脚（见板级引脚表）。

## When to Use

**Applicable:**
- 基于 ESP32（P4/S3/S31/C3/C5/C6/C61）采集摄像头数据流
- 用 V4L2 POSIX API 实现 capture / streaming / 单帧抓拍
- 接入 MIPI-CSI、DVP、SPI、USB-UVC 接口的相机传感器
- 实现图像存储（SD 卡 / SPI Flash / USB MSC）、视频服务器（HTTP MJPEG）、USB 摄像头设备（UVC gadget）
- 启用 ISP Pipeline 处理 RAW 传感器、配置 JPEG/H.264 硬件编解码
- 移植/调试某个具体 sensor 驱动（SC2336、OV5640、OV2640 等 27 款）

**Not applicable:**
- 非 ESP32 芯片的摄像头开发（如树莓派、STM32）
- 纯 PCB 硬件设计、原理图、MIPI 走线阻抗
- esp-iot-solution 中基于旧 `esp32-camera` 组件的工程（API 不同）
- 通用 ESP-IDF 问题与摄像头无关的部分

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先阅读对应 recipe**，其中包含完整调用链、分步代码与常见错误。

### 系统初始化与基础采集

| recipe | scenario |
|---|---|
| `recipes/video_init.md` | 初始化 esp_video 系统：填充 `esp_video_init_config_t`、调用 `esp_video_init()`、板级 SCCB/引脚配置 |
| `recipes/capture_stream.md` | 用 V4L2 POSIX 完整采集流程（S_FMT → REQBUFS → QUERYBUF → mmap → STREAMON → DQBUF） |

### 接口与传感器适配

| recipe | scenario |
|---|---|
| `recipes/mipi_csi.md` | MIPI-CSI 摄像头（ESP32-P4）配置与采集 |
| `recipes/dvp_sensor.md` | DVP 并行接口摄像头（ESP32-P4/S3）引脚与时钟配置 |
| `recipes/spi_sensor.md` | SPI 接口摄像头（C3/C5/C6/C61/P4/S3）含双 SPI 摄像头 |
| `recipes/usb_uvc_host.md` | 作为 USB Host 接入 UVC USB 摄像头 |
| `recipes/custom_format.md` | 用自定义寄存器序列 + `VIDIOC_S_SENSOR_FMT` 初始化非内置格式 |

### 编解码、存储与服务

| recipe | scenario |
|---|---|
| `recipes/jpeg_h264_codec.md` | JPEG / H.264 硬件编解码 M2M 设备使用 |
| `recipes/image_storage.md` | 抓拍存 SD 卡 / SPI Flash / USB MSC |
| `recipes/simple_video_server.md` | 本地 HTTP 服务器提供 MJPEG 视频流与抓拍 |
| `recipes/uvc_gadget.md` | 把 ESP32 实现成 USB UVC 摄像头设备 |
| `recipes/isp_pipeline.md` | 启用 ISP Pipeline Controller 自动 AE/AWB/AF |

---

## 芯片 × 接口 支持矩阵（权威）

来源：`docs/en/Get_Started/index.rst` + `esp_video/idf_component.yml`。

| SoC | MIPI-CSI | DVP | USB-UVC | SPI | JPEG 编解码 | H.264 | ISP |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| ESP32-P4 | ✔ | ✔ | ✔ | ✔ | ✔（依赖 SOC_JPEG_CODEC_SUPPORTED） | ✔ | ✔ |
| ESP32-S3 | — | ✔ | ✔ | ✔ | — | — | — |
| ESP32-S31 | — | — | ✔ | ✔ | ✔ | — | — |
| ESP32-C3 | — | — | — | ✔ | — | — | — |
| ESP32-C5 | — | — | — | ✔ | — | — | — |
| ESP32-C6 | — | — | — | ✔ | — | — | — |
| ESP32-C61 | — | — | — | ✔ | — | — | — |

> DVP 需要 `SOC_LCDCAM_CAM_SUPPORTED`，MIPI-CSI 需要 `SOC_MIPI_CSI_SUPPORTED`，USB-UVC 需要 `SOC_USB_OTG_SUPPORTED`，ISP 需要 `SOC_ISP_SUPPORTED`。这些在 `esp_video/Kconfig` 中由 `depends on` 强约束。

## 视频设备节点速查

来源：`esp_video/include/esp_video_device.h`。

| 设备 | 节点路径 | ID | 用途 |
|---|---|---|---|
| MIPI-CSI | `/dev/video0` | 0 | MIPI 摄像头采集 |
| ISP（DVP 路径） | `/dev/video1` | 1 | DVP 经 ISP 的 RAW 处理 |
| DVP | `/dev/video2` | 2 | 并行接口采集 |
| SPI CAM0 | `/dev/video3` | 3 | SPI 摄像头 0 |
| SPI CAM1 | `/dev/video4` | 4 | 第二路 SPI（需 `ESP_VIDEO_ENABLE_THE_SECOND_SPI_VIDEO_DEVICE`） |
| USB-UVC | `/dev/video40`~`/dev/video49` | 40-49 | USB UVC 摄像头（最多 10 个） |
| JPEG 编码 | `/dev/video10` | 10 | 硬件 JPEG 编码 |
| H.264 编码 | `/dev/video11` | 11 | 硬件 H.264 编码（仅 P4） |
| JPEG 解码 | `/dev/video12` | 12 | 硬件 JPEG 解码 |
| ISP | `/dev/video20` | 20 | ISP 处理设备 |

## 仓库内置示例一览

来源：`esp_video/examples/README.md`。

| 示例路径 | 说明 |
|---|---|
| `esp_video/examples/capture_stream` | 打开视频设备并采集图像数据，遍历所有支持的格式/分辨率/帧率 |
| `esp_video/examples/image_storage/sd_card` | 将 JPEG/H.264 编码后的图像与视频存到 SD 卡 |
| `esp_video/examples/image_storage/usb_msc` | 将图��存到 SPI Flash 并以 USB MSC 暴露给主机 |
| `esp_video/examples/simple_video_server` | 多端口 HTTP 服务器：抓拍、MJPEG 流、相机参数配置 |
| `esp_video/examples/uvc` | 把 ESP32 实现为 USB UVC 摄像头设备 |
| `esp_video/examples/v4l2_cmd` | 类 `v4l2-utils` 命令行：查能力、设控制、抓帧、M2M 转换、ISP 参数 |
| `esp_video/examples/video_custom_format` | 用自定义寄存器序列与 `VIDIOC_S_SENSOR_FMT` 初始化格式 |
| `esp_video/examples/common_components/example_video_common` | 板级初始化公共组件（引脚表见其 README） |

---

## Critical Pitfalls (Must Read)

以下是最常见错误，违反任意一条都会导致采集失败或无图像。

### 1. 必须先 `esp_video_init` 再打开设备节点

```c
// ❌ WRONG — 直接 open 设备，但底层子设备未注册
int fd = open("/dev/video0", O_RDONLY);

// ✅ CORRECT
esp_video_init_config_t cfg = { .csi = &csi_config };
esp_video_init(&cfg);              // 注册 /dev/video0 等
int fd = open("/dev/video0", O_RDONLY);
```

### 2. 必须按 S_FMT → REQBUFS → QUERYBUF → mmap → QBUF → STREAMON 顺序

```c
// ❌ WRONG — 未申请缓冲就 STREAMON
ioctl(fd, VIDIOC_STREAMON, &type);

// ✅ CORRECT
struct v4l2_format fmt = { .type = type,
    .fmt.pix = { .width = w, .height = h, .pixelformat = V4L2_PIX_FMT_RGB565 } };
ioctl(fd, VIDIOC_S_FMT, &fmt);

struct v4l2_requestbuffers req = { .count = 2, .type = type, .memory = V4L2_MEMORY_MMAP };
ioctl(fd, VIDIOC_REQBUFS, &req);

for (int i = 0; i < 2; i++) {
    struct v4l2_buffer buf = { .type = type, .memory = V4L2_MEMORY_MMAP, .index = i };
    ioctl(fd, VIDIOC_QUERYBUF, &buf);
    buffers[i] = mmap(NULL, buf.length, PROT_READ|PROT_WRITE, MAP_SHARED, fd, buf.m.offset);
    ioctl(fd, VIDIOC_QBUF, &buf);
}
ioctl(fd, VIDIOC_STREAMON, &type);
```

### 3. DQBUF 后必须把缓冲重新 QBUF 回去

```c
// ❌ WRONG — 只 DQBUF 不 QBUF，几帧后队列耗尽采集停摆
while (1) { ioctl(fd, VIDIOC_DQBUF, &buf); /* process */ }

// ✅ CORRECT — 处理完归还
while (1) {
    ioctl(fd, VIDIOC_DQBUF, &buf);
    /* process buffer[buf.index] */
    ioctl(fd, VIDIOC_QBUF, &buf);   // 归还
}
```

### 4. 出错帧（V4L2_BUF_FLAG_ERROR）要跳过但仍需归还

```c
// ❌ WRONG — 出错帧直接 break，缓冲泄漏
if (ioctl(fd, VIDIOC_DQBUF, &buf) != 0) break;

// ✅ CORRECT（来自 capture_stream_main.c）
if (ioctl(fd, VIDIOC_DQBUF, &buf) != 0) {
    ESP_LOGE(TAG, "failed to receive video frame");
    return ESP_FAIL;
}
if (buf.flags & V4L2_BUF_FLAG_DONE) {
    frame_size += buf.bytesused;   // 仅统计正常帧
    frame_count++;
}
ioctl(fd, VIDIOC_QBUF, &buf);      // 无论是否出错都归还
```

### 5. SCCB I2C 不能被初始化两次

```c
// ❌ WRONG — init_sccb=true 又传 i2c_handle，冲突
i2c_new_master_bus(&bus_cfg, &bus_handle);
csi_config.sccb_config.init_sccb = true;
csi_config.sccb_config.i2c_handle = bus_handle;

// ✅ CORRECT — 二选一
// 方式A：交给 esp_video 初始化
csi_config.sccb_config.init_sccb = true;
csi_config.sccb_config.i2c_config = { .port = 0, .scl_pin = 8, .sda_pin = 7 };

// 方式B：应用自建后只传 handle
i2c_new_master_bus(&bus_cfg, &bus_handle);
csi_config.sccb_config.init_sccb = false;
csi_config.sccb_config.i2c_handle = bus_handle;
```

### 6. RAW 传感器不启用 ISP 会得到坏图/黑图

```c
// ❌ WRONG — SC2336 RAW8 不开 ISP Pipeline，无颜色还原
// menuconfig: ESP_VIDEO_ENABLE_ISP_VIDEO_DEVICE=y 但 ESP_VIDEO_ENABLE_ISP_PIPELINE_CONTROLLER=n

// ✅ CORRECT — menuconfig 路径
// Component config -> Espressif Video Configuration ->
//   Enable ISP based Video Device -> [*] Enable ISP Pipeline Controller
// 然后 esp_video_init 时带上 ISP flag（esp_video_init 默认 ALL）
```

### 7. `reset_pin`/`pwdn_pin` 无硬件引脚必须填 -1

```c
// ❌ WRONG — 板子无 reset，却填了 0，误把 GPIO0 拉低
.reset_pin = 0,

// ✅ CORRECT
.reset_pin = -1,   // 无则 -1
.pwdn_pin  = -1,
```

### 8. 设备路径宏不要写死字符串

```c
// ❌ WRONG — 硬编码字符串，移植性差
int fd = open("/dev/video0", O_RDONLY);

// ✅ CORRECT — 用 esp_video_device.h 提供的宏
int fd = open(ESP_VIDEO_MIPI_CSI_DEVICE_NAME, O_RDONLY);
// 或 EXAMPLE_CAM_DEV_PATH（example_video_common 自动按所选接口赋值）
```

### 9. SENSOR 控制类要区分 ctrl_class

```c
// ❌ WRONG — 翻转用错了控制类
control[0].id = V4L2_CID_VFLIP;
controls.ctrl_class = V4L2_CTRL_CLASS_ESP_CAM_IOCTL;  // 错

// ✅ CORRECT — VFLIP/HFLIP 属于 USER 类
controls.ctrl_class = V4L2_CTRL_CLASS_USER;
control[0].id = V4L2_CID_VFLIP;
control[0].value = 1;
ioctl(fd, VIDIOC_S_EXT_CTRLS, &controls);

// 读芯片 ID / 寄存器才用 ESP_CAM_IOCTL 类
controls.ctrl_class = V4L2_CTRL_CLASS_ESP_CAM_IOCTL;
control[0].id = ESP_CAM_SENSOR_IOC_G_CHIP_ID;
control[0].p_u8 = (uint8_t *)&chip_id;
control[0].size = sizeof(chip_id);
ioctl(fd, VIDIOC_G_EXT_CTRLS, &controls);
```

### 10. M2M 编解码必须同时管理 OUTPUT 与 CAPTURE 两路缓冲

```c
// ❌ WRONG — 只对一路 REQBUFS，JPEG 设备无法工作

// ✅ CORRECT — 见 image_storage/uvc 示例
// 输入（原始帧）：V4L2_BUF_TYPE_VIDEO_OUTPUT
// 输出（压缩流）：V4L2_BUF_TYPE_VIDEO_CAPTURE
// 各自 REQBUFS/QUERYBUF/QBUF/STREAMON，DQBUF 输入→填数据→QBUF 输出
```

### 11. SPI 视频设备默认在 P4/S3 上是关闭的

```c
// ❌ WRONG — P4 上直接用 /dev/video3，但未开 Kconfig
// menuconfig 漏选 ESP_VIDEO_ENABLE_SPI_VIDEO_DEVICE

// ✅ CORRECT — menuconfig
// Component config -> Espressif Video Configuration -> [*] Enable SPI based Video Device
// 默认在 C3/C5/C6/C61 为 y，P4/S3/S31 默认 n
```

### 12. 第二路 SPI 需显式启用并各自独立 I2C

```c
// ❌ WRONG — 两路 SPI 摄像头共用同一 I2C slave 地址却共用同一 I2C 端口

// ✅ CORRECT — menuconfig:
//   ESP_VIDEO_ENABLE_SPI_VIDEO_DEVICE=y
//   ESP_VIDEO_ENABLE_THE_SECOND_SPI_VIDEO_DEVICE=y
// 且 CAM0/CAM1 使用不同 I2C port（见 common README Note 2）
```

### 13. MIPI-CSI 需要的 XCLK 要单独分配

```c
// ❌ WRONG — 某些传感器（如 ESP32-P4-EYE）需要外部 XCLK，却没启动

// ✅ CORRECT（example_init_video.c）
esp_cam_sensor_xclk_handle_t xclk_handle;
esp_cam_sensor_xclk_config_t xclk_cfg = {
    .esp_clock_router_cfg = { .xclk_pin = 11, .xclk_freq_hz = 24000000 }
};
esp_cam_sensor_xclk_allocate(ESP_CAM_SENSOR_XCLK_ESP_CLOCK_ROUTER, &xclk_handle);
esp_cam_sensor_xclk_start(xclk_handle, &xclk_cfg);
esp_video_init(&cfg);
```

### 14. USERPTR 模式缓冲必须对齐到 PSRAM 缓存行

```c
// ❌ WRONG — 普通malloc，未对齐，DMA 访问异常
buffer[i] = malloc(buf.length);

// ✅ CORRECT（capture_stream Kconfig EXAMPLE_VIDEO_BUFFER_TYPE_USER）
buffer[i] = heap_caps_aligned_alloc(64, buf.length,
                                    MALLOC_CAP_SPIRAM | MALLOC_CAP_CACHE_ALIGNED);
buf.m.userptr = (unsigned long)buffer[i];
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 确认目标芯片、相机接口（MIPI/DVP/SPI/USB）、sensor 型号、输出格式（RAW/YUV/RGB/JPEG） |
| 2 | Recipe | 在 `recipes/` 中匹配场景，按其调用链实现 |
| 3 | Query | recipe 未覆盖的 API 查 `resources/api_reference.md`，配置项查 `resources/config_reference.md` |
| 4 | Validate | 核对：设备节点宏、ioctl 顺序、SCCB 配置、Kconfig 是否启用对应 video device、引脚表 |
| 5 | Confirm | 向用户给出方案：includes、`esp_video_init_config_t` 字段、采集循环、menuconfig 选项 |
| 6 | Execute | 新工程：`idf.py add-dependency esp_video`，复制最近示例修改；已有工程：原地编辑 |
| 7 | Build | `idf.py set-target esp32p4` → `idf.py menuconfig`（开对应 video device / ISP / codec）→ `idf.py build` |
| 8 | Flash | `idf.py -p PORT flash monitor` |
| 9 | Debug | 看启动日志的 `driver/card/bus_info`（VIDIOC_QUERYCAP）、chip id、帧计数 FPS |

### Step 6 Detail — 工程创建策略

**新工程首选就近复制官方示例**（位于 `esp_video/examples/`），再按目标芯片/接口修改 `menuconfig`：

- 通用采集/性能评估 → `capture_stream`
- 存图/存视频 → `image_storage/sd_card` 或 `image_storage/usb_msc`
- 局域网视频预览 → `simple_video_server`
- 把设备做成 USB 摄像头 → `uvc`
- 调试 V4L2 控制/ISP 参数 → `v4l2_cmd`
- 自定义传感器寄存器序列 → `video_custom_format`

板级引脚与 SCCB 配置由 `common_components/example_video_common` 统一管理，菜单选板子或 `Customized Development Board` 自填引脚（引脚表见 `resources/example_list.md`）。

---

## Failure Strategies

| Situation | Action |
|---|---|
| `open("/dev/videoN")` 返回 -1 | 检查是否调用 `esp_video_init`；检查对应 Kconfig（MIPI/DVP/SPI/USB）是否启用 |
| `VIDIOC_S_FMT` 失败 | 格式/分辨率不被 sensor 支持，用 `VIDIOC_ENUM_FMT`+`VIDIOC_ENUM_FRAMESIZES` 枚举真实支持项 |
| 采集几帧后停摆 | DQBUF 后未 QBUF 归还缓冲（pitfall 3） |
| RAW 传感器画面发紫/偏色/黑 | 未启用 ISP Pipeline Controller（pitfall 6） |
| SCCB 报错或 sensor 探测失败 | I2C 引脚/端口/频率不对；或 `init_sccb` 与 `i2c_handle` 冲突（pitfall 5） |
| SPI 摄像头在 P4 上不可用 | 默认关闭，需 menuconfig 启用 `ESP_VIDEO_ENABLE_SPI_VIDEO_DEVICE`（pitfall 11） |
| H.264 编译报错 | 仅 ESP32-P4 支持，且需 `esp_h264` 依赖（rules: target=esp32p4） |
| UVC gadget 无图像 | 检查编码器（JPEG/H264）是否打开、M2M 两路缓冲是否齐备（pitfall 10） |
| API 在 resources 中查不到 | 停止并告知用户该 API 不存在于本仓库，不要臆造 |

## References

- 场景 recipes → `recipes/` 目录
- V4L2 / esp_video / esp_cam_sensor API 速查 → `resources/api_reference.md`
- Kconfig 配置项速查 → `resources/config_reference.md`
- 综合陷阱清单 → `resources/pitfalls.md`
- 示例与板级引脚一览 → `resources/example_list.md`
- 仓库原文：`docs/en/Get_Started/index.rst`、`esp_video/include/*`、`esp_cam_sensor/include/*`、`esp_video/examples/*`
