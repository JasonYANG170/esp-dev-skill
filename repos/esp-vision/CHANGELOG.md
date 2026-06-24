# Changelog

本文件记录 esp-vision-skill 的变更。格式参考 [Keep a Changelog](https://keepachangelog.com/)。

## [1.1.0] - 2026-06-18

补齐审计确认的 4 个配方空缺：Wi-Fi MJPEG 推流、云端 AI 视觉推理、帧差法运动检测、绘图原语总览。所有 API、示例路径与签名均来自仓库真实来源。

### 新增
- `recipes/wifi_mjpeg_stream.md` — Wi-Fi HTTP MJPEG 流（`multipart/x-mixed-replace`），ESP32-S3 等无 H.264 板的主推流路径。来源示例 `example/01-Camera/03-MJPEG/wifi_mjpeg_stream.py`；概念文档 `docs/zh_CN/concepts/codec-streaming.rst`。
- `recipes/cloud_ai_vision.md` — 采集帧 JPEG+base64 经 HTTPS POST 到 OpenAI 兼容 vision API，含 TLS/`ssl`、chunked 解码、按键边沿触发。来源示例 `example/07-Network/01-Cloud-AI/openai_compatible_vision.py`。
- `recipes/frame_differencing.md` — `image.Image.difference()` + `binary()` 背景差分运动检测，及内存 `image.ImageIO((w,h,fmt), count)` 帧历史。来源示例 `example/02-Image-Processing/03-Frame-Differencing/in_memory_frame_differencing.py`；文档 `docs/zh_CN/api-reference/imageio.rst`。
- `recipes/image_drawing.md` — `draw_line/rectangle/circle/ellipse/cross/arrow/string/image` 总览，RGB565 vs GRAYSCALE 颜色约定。来源示例 `example/02-Image-Processing/00-Drawing/{drawing,shape_drawing,line_drawing}.py`。

### 变更
- `SKILL.md`：`metadata.version` 1.0.0 → 1.1.0；在 Scenario Quick Reference 新增「网络与云端」分组，图像处理与显示/推流分组各补条目。
- `resources/api_reference.md`：新增「Wi-Fi MJPEG / 云端 AI」一节，收录 `to_jpeg`/`bytearray` 用法、HTTP multipart 帧格式、`ssl.SSLContext` + `CERT_NONE` 与 chunked 解码约定。

## [1.0.0] - 2026-06-18

首个版本。基于 ESP-VISION 仓库的 `docs/`、`stubs/*.pyi`、`boards/`、`micropython.cmake`、`Makefile` 与 `example/` 编写，所有 API、类、常量、配置宏与文件路径均来自仓库真实来源。

### 新增
- `SKILL.md`：12 条核心原则、何时用、配方索引（按类别分组）、开发板/芯片能力表、ESP-IDF 兼容性表、关键模块速查、12 条避坑（含错误/正确代码对照）、执行工作流、失败策略、参考。
- `AGENTS.md`：项目上下文、命名/import/骨架模式、产品入口模式、资源释放模式、构建工作流、脚本 codegen 清单、Do Not Modify 说明。
- `recipes/`：12 个场景配方
  - `hello_camera.md`（首个摄像头脚本）
  - `camera_snapshot_save.md`（采集并保存图像）
  - `camera_orientation.md`（镜像/翻转/`status()` 诊断）
  - `color_blob_tracking.md`（LAB 阈值色块追踪）
  - `image_filters.md`（滤波与二值化）
  - `qrcode_barcode_apriltag.md`（二维码/条码/AprilTag）
  - `object_detection.md`（ESPDet / YOLO11）
  - `pose_estimation.md`（YOLO11nPose 姿态）
  - `image_classification.md`（ImageNetCls 分类）
  - `lcd_display.md`（板载 LCD 显示）
  - `h264_record.md`（H.264 录制，仅 P4）
  - `rtsp_stream.md`（RTSP 推流，仅 P4）
  - `asyncio_pipeline.md`（asyncio 多协程应用）
  - `customize_firmware.md`（客制化固件）
- `resources/`：
  - `api_reference.md`（按模块分组的真实签名，源自 `stubs/*.pyi`）
  - `config_reference.md`（构建变量、`boardconfig.h`、`imlib_config.h`、`board.cmake`、`mpconfigboard.h`、`manifest.py`）
  - `pitfalls.md`（16 条避坑）
  - `example_list.md`（`example/` 全部脚本索引）
- `README.md`：中文介绍、功能、安装、支持范围、目录结构。
