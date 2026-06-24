---
name: esp-vision-skill
description: >-
  AI Skill for ESP-VISION (乐鑫低代码端侧 AI 与计算机视觉框架) 的 MicroPython 脚本与固件开发。
  适用于在 ESP32-P4 / ESP32-S3 / ESP32-S31 开发板上编写摄像头采集、图像处理、ESP-DL 模型推理、
  LCD 显示、H.264 编码、RTSP 推流等应用，以及客制化 ESP-VISION 固件、添加模型与自定义 Python 模块。
  Trigger words: "ESP-VISION", "ESP32-P4", "ESP32-S3", "ESP32-S31", "ESP32_P4X_EYE", "ESP32_S3_EYE",
  "esp-vision", "乐鑫视觉", "端侧 AI", "edge AI", "computer vision", "计算机视觉", "ESP-DL",
  "espdl", "MicroPython", "imlib", "OpenMV", "sensor.snapshot", "图像处理", "摄像头", "目标检测", "姿态估计"
tags:
  - embedded
  - esp32
  - esp-vision
  - computer-vision
  - edge-ai
  - esp-dl
  - micropython
  - imlib
  - openmv
  - camera
  - h264
  - rtsp
license: Apache-2.0
compatibility: >-
  Target chips ESP32-P4 / ESP32-S3 / ESP32-S31; builds with ESP-IDF release/v5.5, release/v6.0, or master
  via the board-aware `idf.py --board <BOARD>` extension (or top-level Makefile). Language: MicroPython v1.28.0.
metadata:
  author: Community
  version: "1.1.0"
---

# esp-vision-skill

AI Skill for ESP-VISION (Low-Code Edge AI & Computer Vision Framework for Espressif SoCs)。本 Skill 面向 ESP-VISION 仓库，提供场景化配方、API 速查、配置参考与避坑要点，帮助 AI agent 正确地为 ESP32-P4 / ESP32-S3 / ESP32-S31 开发板编写 MicroPython 视觉脚本，并客制化 ESP-VISION 固件、引入 `.espdl` 模型或新增 Python 模块。所有 API 名称、类、常量、配置宏与文件路径均来自仓库真实文档 (`docs/`)、类型存根 (`stubs/*.pyi`)、板级配置 (`boards/`) 与示例脚本 (`example/`)，不存在则不写。

## Core Principles

1. **API 来源唯一** — ESP-VISION 的 Python 模块只有 `sensor`、`image`、`display`、`espdl`，外加随芯片启用的 `h264`、`rtsp`，以及 `image.ImageIO` 类型。查 `resources/` 没有对应条目即视为该 API 不存在，绝不臆造。
2. **OpenMV 兼容命名** — 摄像头模块叫 `sensor`（不是 `camera`）；视觉算法来自 OpenMV 的 `imlib`（仓库内 `components/imlib`，MIT 维护）。大多数 OpenMV 脚本可直接移植。
3. **采集流水线初始化顺序固定** — `sensor.reset()` → `set_pixformat()` → `set_framesize()` → `skip_frames(time=...)` 等待自动曝光/白平衡稳定 → `sensor.snapshot()`。缺 `skip_frames` 会导致前几帧曝光异常。
4. **`sensor.snapshot()` 返回可复用帧缓冲** — 返回的 `image.Image` 由共享帧缓冲承载；下一帧采集会覆盖它。需要在采集后继续使用当前帧时，必须 `.copy()`。
5. **像素格式与分辨率只有两档常量** — `set_pixformat` 接受 `sensor.GRAYSCALE` / `sensor.RGB565`；`set_framesize` 接受 `sensor.QQVGA` / `sensor.QVGA`（来源 `stubs/sensor.pyi`）。不要假设 OpenMV 全部分辨率都可用。
6. **`h264` 与 `rtsp` 仅 ESP32-P4 构建** — `micropython.cmake` 仅在 `IDF_TARGET == esp32p4` 时加入这两个模块。ESP32-S3 脚本应改用 MJPEG（JPEG 流）。
7. **模型不在固件中，运行时从存储加载** — `.espdl` 模型放在 `/sdcard/...` 或 `/flash/...`，用 `espdl.ESPDet` / `YOLO11` / `YOLO11nPose` / `ImageNetCls` 封装类构造，`detect()`/`classify()` 返回的是元组列表。
8. **模型只加载一次并跨帧复用** — 模型权重与激活缓冲分配在 PSRAM；逐帧重建封装对象会大幅拖慢并耗尽内存。用完调用 `deinit()`。
9. **编码器尺寸必须与每帧输入一致** — `h264.H264Encoder(width, height, ...)` 创建后，每帧 `encode()` 的图像尺寸必须与构造尺寸一致；分辨率变了要重建编码器。
10. **主机预览用 `img.flush()`** — 开发期最快的反馈回路是 `img.flush()`，经 USB CDC 推送 EVFRAME JPEG，无需 Wi-Fi、无需 SD 卡。
11. **资源用 `try/finally` 释放** — `espdl.*.deinit()`、`h264.H264Encoder.close()`、`rtsp.RTSPServer.stop()`、文件 `close()` 都应在 `finally` 中调用，避免异常时泄漏。
12. **不要启用 GPL 代码路径** — 构建固定 `OMV_NO_GPL=1`；未经许可证审查不得移除。每块板的 `imlib_config.h`（`IMLIB_ENABLE_*`）决定编译哪些可选算法，未启用的方法调用时会抛异常。

## When to Use

**Applicable:**
- 为 ESP-VISION 支持的开发板编写 MicroPython 摄像头采集脚本（`sensor`）
- 在采集帧上做图像处理（绘图、滤波、二值化、色块追踪、统计、形态学）
- 检测二维码、1D/2D 条码、AprilTag、直线、圆、矩形等特征
- 运行 ESP-DL `.espdl` 模型：目标检测（ESPDet / YOLO11）、姿态估计（YOLO11nPose）、图像分类（ImageNetCls）
- 把画面输出到板载 LCD（`display`）或经 `img.flush()` 推到主机预览
- 录制 / 回放图像序列（`image.ImageIO`），或编码 H.264、RTSP 推流（ESP32-P4）
- 客制化 ESP-VISION 固件：调整 `micropython.cmake`、`imlib_config.h`、`manifest.py`、`mpconfigboard.h`
- 引入新 `.espdl` 模型，或通过 `USER_C_MODULES` 新增 Python 模块

**Not applicable:**
- 裸 ESP-IDF C/C++ 摄像头应用（ESP-VISION 是 MicroPython 运行时；若要写纯 C/IDF 应用，请用 `esp32-camera` / `esp_video`，不属于本 Skill 范围）
- 其它厂商芯片（如 K210、RK 系列）
- PCB 布线、摄像头模组硬件选型与原理图设计
- 训练 / 量化模型本身（ESP-VISION 只负责部署与推理；模型转换用 ESP-DL 工具链，参见 `how-to/add-model.rst`）

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读对应配方** — 配方包含完整的调用链、分步说明、真实代码与常见错误表。

### 入门与采集

| 配方 | 场景 |
|---|---|
| `recipes/hello_camera.md` | 首个脚本：复位传感器、选格式/分辨率、连续采集并 `flush` 预览 |
| `recipes/camera_snapshot_save.md` | 采集一帧并保存为 jpg/bmp/ppm 到 SD 卡 |
| `recipes/camera_orientation.md` | 水平镜像 / 垂直翻转校正安装方向，用 `status()` 诊断 |

### 图像处理

| 配方 | 场景 |
|---|---|
| `recipes/image_drawing.md` | 绘图原语总览（draw_line/rectangle/circle/ellipse/cross/arrow/string），RGB565 vs GRAYSCALE 颜色约定 |
| `recipes/color_blob_tracking.md` | LAB 阈值 + `find_blobs` 色块追踪与标注 |
| `recipes/image_filters.md` | 滤波（mean/gaussian/median/bilateral/laplacian/histeq）与二值化 |
| `recipes/frame_differencing.md` | 帧差法运动检测：`difference` + `binary`，内存 `image.ImageIO` 帧历史 |
| `recipes/qrcode_barcode_apriltag.md` | 二维码、条码（P4 ZXing）、AprilTag 检测 |

### AI 推理（ESP-DL）

| 配方 | 场景 |
|---|---|
| `recipes/object_detection.md` | ESPDet / YOLO11 目标检测，绘制检测框并运行时调阈值 |
| `recipes/pose_estimation.md` | YOLO11nPose 17 关键点姿态估计与骨架绘制 |
| `recipes/image_classification.md` | ImageNetCls 图像分类（文件图分类） |

### 显示、编码与推流

| 配方 | 场景 |
|---|---|
| `recipes/lcd_display.md` | 板载 LCD 显示：`display.Display().write()`、缩放、ROI、背光 |
| `recipes/h264_record.md` | （P4）H.264 录制裸码流到 SD 卡 |
| `recipes/rtsp_stream.md` | （P4）以太网 + H.264 + RTSP 推流 |
| `recipes/wifi_mjpeg_stream.md` | （S3/通用）Wi-Fi HTTP MJPEG 流，浏览器直接观看（无 H.264 时的主推流路径） |

### 网络与云端

| 配方 | 场景 |
|---|---|
| `recipes/cloud_ai_vision.md` | 采集帧 JPEG+base64 经 HTTPS POST 到 OpenAI 兼容 vision API，打印描述 |

### 工程化与固件

| 配方 | 场景 |
|---|---|
| `recipes/asyncio_pipeline.md` | 用 MicroPython asyncio 组织采集/推理/遥测多协程应用 |
| `recipes/customize_firmware.md` | 客制化固件功能（模块 / imlib / MicroPython / 冻结代码 / 板级） |

---

## 支持的开发板与芯片能力

ESP-VISION 文档按芯片分别构建。摄像头后端在构建时按板选择，公开 `sensor` API 在各板一致。

| 开发板 (`--board`) | 芯片 | 摄像头后端 | H.264/RTSP | 条码（ZXing-C++） |
|---|---|---|---|---|
| `ESP32_P4X_EYE` | ESP32-P4 | `esp_video` / V4L2 + PPA（MIPI-CSI） | 是 | 是 |
| `ESP32_P4X_FUNCTION_EV_BOARD` | ESP32-P4 | `esp_video` / V4L2 + PPA | 是 | 是 |
| `ESP32_S3_EYE` | ESP32-S3 | `esp32-camera`（DVP，软件转换） | 否 | 否 |
| `ESP32_S31_KORVO` | ESP32-S31 | `esp_video` / V4L2（需 IDF master overlay） | 否 | 否 |
| `TEMPLATE` | — | — | — | —（新板 bring-up 用） |

> 来源：`README.md`、`docs/zh_CN/concepts/camera-pipeline.rst`、`boards/*/port/board.json`。`micropython.cmake`、`boards/*/port/mpconfigboard.cmake` 与各 `board.cmake` 是 API 可用性的权威来源。

## ESP-IDF 兼容性

| ESP-IDF 分支 | 支持 | 备注 |
|---|---|---|
| `release/v5.5` | 是 | 功能最完整：SSL/TLS、`cryptolib`、WebSocket、WebREPL、socket events、`machine.I2S`、PCNT、I2C target。需要这些可选功能时推荐。 |
| `release/v6.0` | 是 | 提供 `network`/WLAN/socket；上述可选功能在当前 overlay 下仍关闭。当前不支持 ESP32-S31 构建。 |
| `master` | 是 | 同 v6.0，可选功能仍关闭。ESP32-S31 必须使用。 |

> 来源：`README.md` ESP-IDF Compatibility 表。构建前需 source IDF `export.sh` 使 `idf.py` 在 `PATH`。

## 关键模块速查

| 模块 | 类型 | 关键类/函数 | 典型用途 |
|---|---|---|---|
| `sensor` | 内置 C 模块 | `reset`, `set_pixformat`, `set_framesize`, `skip_frames`, `snapshot`, `set_hmirror`, `set_vflip`, `shutdown`, `status` | 摄像头采集 |
| `image` | 内置 C 模块 | `Image` 类、`ImageIO`、`find_blobs`/`find_qrcodes`/`find_apriltags`/`find_lines`/`find_circles`、绘图与滤波方法 | 图像处理 + 编解码 |
| `display` | 内置 C 模块 | `ESP32Display`（别名 `Display`）：`write`, `width`, `height`, `backlight`, `clear`, `deinit` | 板载 LCD |
| `espdl` | 内置 C++ 模块 | `load_model`, `ESPDet`, `YOLO11`, `YOLO11nPose`, `ImageNetCls` | ESP-DL 推理 |
| `h264`（仅 P4） | 内置 C 模块 | `H264Encoder`：`encode`, `keyframe`, `close` | H.264 编码 |
| `rtsp`（仅 P4） | 内置 C 模块 | `RTSPServer`：`send`, `stop` | RTSP 推流 |

> 完整签名见 `resources/api_reference.md`。

---

## Critical Pitfalls (Must Read)

下列是最常见的错误，违反任何一条都会导致脚本不可用或性能严重下降。

### 1. 缺少 `skip_frames` 导致前几帧曝光异常

```python
# ❌ WRONG — 复位后立即采集，自动曝光/白平衡未收敛
sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
img = sensor.snapshot()  # 偏色、过曝

# ✅ CORRECT — 配置后丢弃若干帧等待稳定
sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)  # 或 sensor.skip_frames(n=3)
img = sensor.snapshot()
```

### 2. `snapshot()` 返回可复用帧缓冲，跨帧使用未复制

```python
# ❌ WRONG — 保存的 background 会被下一帧覆盖
background = sensor.snapshot()
while True:
    img = sensor.snapshot()
    img.difference(background)  # background 已被覆盖，差分无效

# ✅ CORRECT — 背景帧显式 copy
background = sensor.snapshot().copy()
while True:
    img = sensor.snapshot()
    img.difference(background)
```

### 3. 在 ESP32-S3 上调用仅 P4 可用的 `h264` / `rtsp`

```python
# ❌ WRONG — S3 固件没有 h264 模块
import h264
enc = h264.H264Encoder(320, 240, fps=15)  # ImportError

# ✅ CORRECT — S3 用 MJPEG（JPEG 流）替代
import sensor
sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)
while True:
    jpg = sensor.snapshot().to_jpeg(quality=50)
    # 经 socket 发送 jpg.bytearray()，见 recipes/mjpeg 与 example/01-Camera/03-MJPEG
```

### 4. 调用未启用的 imlib 算法

```python
# ❌ WRONG — 该板 imlib_config.h 未定义对应 IMLIB_ENABLE_*，调用抛异常
img.find_apriltags(...)  # 若未定义 IMLIB_ENABLE_APRILTAGS

# ✅ CORRECT — 先确认板级 imlib_config.h 启用，否则换板或改配置
# boards/ESP32_P4X_EYE/imlib_config.h 已启用 IMLIB_ENABLE_APRILTAGS / TAG36H11 等
img = sensor.snapshot()
for tag in img.find_apriltags(families=image.TAG36H11, pose=False):
    img.draw_rectangle(tag.rect, color=255, thickness=2)
```

### 5. 逐帧重建 ESP-DL 模型（内存/性能）

```python
# ❌ WRONG — 每帧重新构造，反复分配 PSRAM 并重新加载权重
while True:
    img = sensor.snapshot()
    det = espdl.ESPDet("/sdcard/model.espdl", score=0.5)  # 极慢
    det.detect(img)

# ✅ CORRECT — 模型构造一次，跨帧复用，finally 中 deinit
det = espdl.ESPDet("/sdcard/model.espdl", score=0.5, nms=0.7)
try:
    while True:
        img = sensor.snapshot()
        for x, y, w, h, score, category in det.detect(img):
            img.draw_rectangle(x, y, w, h, color=(255, 0, 0))
        img.flush()
finally:
    det.deinit()
```

### 6. H.264 编码器尺寸与输入帧不一致

```python
# ❌ WRONG — 固定 320x240，但相机改了分辨率
enc = h264.H264Encoder(320, 240, fps=15)
sensor.set_framesize(sensor.QVGA)  # 假设输出非 320x240 会抛错

# ✅ CORRECT — 用首帧尺寸构造，分辨率变了重建编码器
img = sensor.snapshot()
enc = h264.H264Encoder(img.width(), img.height(), fps=15)
```

### 7. 资源未在 `finally` 中释放

```python
# ❌ WRONG — 异常时 server / encoder 泄漏
server = rtsp.RTSPServer(WIDTH, HEIGHT, fps=FPS, listen_port=8554)
while True:
    server.send(enc.encode(sensor.snapshot()))  # Ctrl-C 直接退出

# ✅ CORRECT — finally 清理
try:
    while True:
        server.send(enc.encode(sensor.snapshot()))
finally:
    server.stop()
    enc.close()
```

### 8. 文件流用完未 `close()`（容器元数据丢失）

```python
# ❌ WRONG — 未 close，容器元数据/缓冲未落盘
stream = image.ImageIO("/sdcard/stream.bin", "w")
for _ in range(30):
    stream.write(sensor.snapshot())

# ✅ CORRECT — 写完后 sync 并 close
stream = image.ImageIO("/sdcard/stream.bin", "w")
for _ in range(30):
    stream.write(sensor.snapshot())
stream.sync()
stream.close()
```

### 9. `display.Display()` 重复创建

```python
# ❌ WRONG — 循环里反复构造，重复初始化屏幕资源
while True:
    lcd = display.Display()
    lcd.write(sensor.snapshot())

# ✅ CORRECT — 显示对象只创建一次并复用
lcd = display.Display()
while True:
    lcd.write(sensor.snapshot())
```

### 10. AprilTag 当成方法调用（其属性是字段而非方法）

```python
# ❌ WRONG — 把 corners/x/y/id 当方法调用
for tag in img.find_apriltags(families=image.TAG36H11):
    print(tag.id(), tag.rect())  # apriltag 的这些是字段，不是方法

# ✅ CORRECT — apriltag 的属性是字段（直接访问）；qrcode/barcode/blob 等才是方法
for tag in img.find_apriltags(families=image.TAG36H11):
    img.draw_rectangle(tag.rect, color=255, thickness=2)
    print("tag:", tag.name, "id:", tag.id, "margin:", tag.decision_margin)
```

### 11. 在 `boot.py` 里放死循环（阻塞 USB REPL）

```python
# ❌ WRONG — boot.py 永不返回，USB REPL 与主机工具无法使用
# /boot.py
while True:
    do_something()

# ✅ CORRECT — boot.py 只做短时确定性初始化后返回
# /boot.py
import network
wlan = network.WLAN(network.STA_IF)
wlan.active(True)
# 产品应用循环应放在 /main.py 里
```

### 12. 误用 CPython 文档判断 MicroPython 行为

```python
# ❌ WRONG — 以 CPython 文档为准，可能与设备实际不符
import time
# 假设 time.time() 行为与 CPython 完全一致

# ✅ CORRECT — 以 MicroPython v1.28.0 文档为准；测量间隔用 ticks_ms/ticks_diff
t0 = time.ticks_ms()
# ... do work ...
elapsed = time.ticks_diff(time.ticks_ms(), t0)
```

> ESP-VISION 固定 MicroPython v1.28.0，应以 `https://docs.micropython.org/en/v1.28.0/` 为权威参考。

---

## Execution Workflow

| 步骤 | 名称 | 说明 |
|------|------|------|
| 1 | 理解需求 | 明确目标芯片/开发板、像素格式/分辨率、是否需要 AI 推理、显示或推流 |
| 2 | 选配方 | 在 `recipes/` 找匹配场景；命中则按其调用链推进 |
| 3 | 查 API | 未被配方覆盖的 API 查 `resources/api_reference.md`；配置项查 `resources/config_reference.md` |
| 4 | 校验可用性 | 确认目标 API 在所选芯片/板上可用（`h264`/`rtsp` 仅 P4；条码仅 P4；分辨率仅 QQVGA/QVGA） |
| 5 | 复用示例 | 优先以 `example/` 下最接近的脚本为起点改写（见 `resources/example_list.md`） |
| 6 | 确认计划 | 向用户说明：import、采集初始化顺序、循环体、资源释放 |
| 7 | 实现 | 写 MicroPython 脚本；遵循 OpenMV 兼容命名（`sensor`、`image`） |
| 8 | 自查 | 检查 `skip_frames`、跨帧 `.copy()`、模型/编码器复用、`try/finally` 释放 |
| 9 | 部署 | `idf.py --board <BOARD> -p <PORT> build flash monitor`；或在 REPL 运行/烧录到 `/main.py` |
| 10 | 调试 | `img.flush()` 主机预览、`print` 到串口、`help("modules")` 查可用模块 |

### 步骤 5 详情 — 脚本起点选择

ESP-VISION 提供编号化的 `example/` 脚本，按场景选择最接近的作为起点：

- 首个脚本 → `example/00-HelloWorld/helloworld.py`
- 采集并保存 → `example/01-Camera/00-Snapshot/save_snapshot.py`
- H.264 录制（P4）→ `example/01-Camera/01-H264/record_h264.py`
- RTSP 推流（P4）→ `example/01-Camera/02-RTSP/stream_rtsp.py`
- Wi-Fi MJPEG（S3/通用）→ `example/01-Camera/03-MJPEG/wifi_mjpeg_stream.py`
- 色块追踪 → `example/02-Image-Processing/02-Color-Tracking/find_blobs.py`
- 滤波 → `example/02-Image-Processing/01-Image-Filters/filters.py`
- 帧差分（内存 ImageIO）→ `example/02-Image-Processing/03-Frame-Differencing/in_memory_frame_differencing.py`
- 目标检测 → `example/03-Machine-Learning/00-ESP-DL/espdet_pico.py` / `yolo11.py`
- 姿态估计 → `example/03-Machine-Learning/00-ESP-DL/yolo11n_pose.py`
- 图像分类 → `example/03-Machine-Learning/00-ESP-DL/imagenet_cls.py`
- 二维码 → `example/04-Barcodes/find_qrcodes.py`
- AprilTag / 直线圆 / 条码 → `example/05-Feature-Detection/*`
- LCD 预览 / SD 卡 / 舵机 → `example/06-Peripherals/*`
- Wi-Fi / 云 AI → `example/07-Network/*`

完整清单见 `resources/example_list.md`。

---

## Failure Strategies

| 情况 | 处理 |
|---|---|
| API 不在 `resources/` 中 | 立即停止，告知用户该 API 不存在（或仅在特定芯片/板上可用） |
| 不确定 `h264`/`rtsp` 是否可用 | 仅 ESP32-P4 构建包含；S3/S31 脚本改用 MJPEG |
| 设备无法启动（坏 `main.py`） | REPL 连接后 `Ctrl-C`，`os.rename("/main.py","/main.disabled.py")` 再 `Ctrl-D` 软复位；必要时 `idf.py --board <BOARD> -p <PORT> erase-flash` 重烧 |
| 摄像头偏色/过曝 | 检查是否调用 `sensor.skip_frames(time=...)` 等待 AE/AWB 收敛 |
| 模型推理 OOM / 慢 | 模型只构造一次；降低采集分辨率；用更小模型（如 `espdet_pico`） |
| 编码帧率达不到 | 降分辨率/帧率/码率，而不是反复创建编码器 |
| RTSP 客户端卡顿 | `send()` 不阻塞，慢客户端会整帧丢弃；必要时调大 `max_frame_len` 或降码率 |
| 某算法调用抛异常 | 检查对应板级 `boards/<BOARD>/imlib_config.h` 是否 `#define IMLIB_ENABLE_*` |
| 模块导入失败 | REPL 执行 `help("modules")` 确认可用性；可能受 `mpconfigboard.h` / IDF 版本裁剪 |

## References

- 场景配方 → `recipes/` 目录
- API 速查 → `resources/api_reference.md`
- 配置参考 → `resources/config_reference.md`
- 避坑汇总 → `resources/pitfalls.md`
- 示例脚本索引 → `resources/example_list.md`
- 上游权威文档（中文）→ https://docs.espressif.com/projects/esp-vision/zh_CN/latest/
- MicroPython v1.28.0 文档 → https://docs.micropython.org/en/v1.28.0/
- 仓库源码（API 权威来源）→ `stubs/*.pyi`、`micropython.cmake`、`boards/<BOARD>/`、`example/`
