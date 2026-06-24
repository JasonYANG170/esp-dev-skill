# Changelog

## [1.0.0] - 2026-06-18

### Added

- 初始版本，面向 Espressif esp-video-components 仓库的 AI 开发技能。
- **SKILL.md**：YAML frontmatter（name/description/tags/license/compatibility/metadata）+ 核心原则（12 条）+ When to Use + 场景速查表（11 个 recipe）+ 芯片×接口支持矩阵 + 视频设备节点速查 + 内置示例一览 + 14 条 Critical Pitfalls（含 WRONG/CORRECT 代码）+ 执行工作流 + 失败策略 + References。
- **AGENTS.md**：工程上下文、文件命名、include 模式、标准工程结构、入口/初始化模式、构建工作流、代码生成 checklist、Do Not Modify 约定。
- **recipes/**（11 个场景）：
  - `video_init.md` — esp_video 系统初始化（csi/dvp/spi/usb_uvc 配置、SCCB 二选一、XCLK）
  - `capture_stream.md` — V4L2 POSIX 完整采集流程（S_FMT→REQBUFS→QUERYBUF→mmap→QBUF→STREAMON→DQBUF）
  - `mipi_csi.md` — MIPI-CSI 摄像头（ESP32-P4）与 ISP Pipeline
  - `dvp_sensor.md` — DVP 并行接口（P4/S3）引脚与时钟
  - `spi_sensor.md` — SPI 接口（全系列）与双 SPI 摄像头
  - `usb_uvc_host.md` — USB Host 接入 UVC 摄像头
  - `custom_format.md` — 自定义寄存器序列 + VIDIOC_S_SENSOR_FMT
  - `jpeg_h264_codec.md` — JPEG/H.264 硬件编解码 M2M 设备
  - `image_storage.md` — 图像/视频存储（SD/Flash/USB MSC）
  - `simple_video_server.md` — 本地 HTTP 视频服务器（抓拍 + MJPEG 流）
  - `uvc_gadget.md` — ESP32 作为 USB UVC 摄像头设备
  - `isp_pipeline.md` — ISP Pipeline 自动 AE/AWB/AF
- **resources/**（4 个速查文档）：
  - `api_reference.md` — esp_video / esp_cam_sensor / esp_sccb_intf / xclk / 公共组件真实 API
  - `config_reference.md` — 全部 Kconfig 选项、依赖、target rules、menuconfig 路径
  - `pitfalls.md` — 11 主题 45 条陷阱
  - `example_list.md` — 全部官方示例、板级引脚表、27 款 sensor 驱动
- **README.md** 与 **CHANGELOG.md**。

### Grounding

- 全部 API、结构体、宏、Kconfig、设备节点、引脚号、代码片段均来自 esp-video-components 仓库真实文件（`esp_video/include/*`、`esp_cam_sensor/include/*`、`esp_video/Kconfig`、`esp_video/idf_component.yml`、`esp_video/examples/*`、`docs/en/*`）。
- 文档较薄的章节（`docs/en/ESP_Camera_Sensor/cam_sensor_driver.rst`、`cam_motor_driver.rst` 为空 stub）改以头文件与示例代码为依据，未臆造内容。
