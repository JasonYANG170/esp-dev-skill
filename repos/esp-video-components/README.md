# esp-video-components-skill

面向 Espressif **esp-video-components** 仓库的 AI 开发技能（Claude Code / Agent Skill 格式）。esp-video-components 是乐鑫官方的摄像头/视频组件集合，提供与 Linux V4L2 标准兼容的 POSIX API，支持 MIPI-CSI、DVP、SPI、USB-UVC 四类相机接口，以及 JPEG/H.264 硬件编解码与 ISP 自动图像处理。

本技能以仓库真实文档、头文件与示例代码为唯一依据，所有 API、结构体、宏、Kconfig、引脚、代码片段均来自源码，不做任何臆造。

## 功能特性

- **场景化 recipes（11 个）**：覆盖系统初始化、V4L2 采集、MIPI/DVP/SPI/USB-UVC 接口、自定义格式、JPEG/H.264 编解码、图像存储、HTTP 视频服务器、UVC gadget、ISP Pipeline。
- **API 速查**：`resources/api_reference.md` 汇总 esp_video、esp_cam_sensor、esp_sccb_intf、xclk、公共组件的真实函数签名与类型。
- **配置速查**：`resources/config_reference.md` 列出全部 Kconfig 选项、依赖关系与 target rules。
- **陷阱清单**：`resources/pitfalls.md` 按 11 个主题汇总 45 条常见错误与对策。
- **示例与引脚一览**：`resources/example_list.md` 列出全部官方示例、板级引脚表与 27 款 sensor 驱动。
- **芯片 × 接口支持矩阵**：明确 ESP32-P4/S3/S31/C3/C5/C6/C61 各自支持的接口与编解码能力。

## 适用范围

- 芯片：ESP32-P4 / S3 / S31 / C3 / C5 / C6 / C61
- 框架：ESP-IDF >= 5.4
- 组件：`esp_video`（ESPRESSIF MIT）、`esp_cam_sensor`（Apache-2.0）、`esp_sccb_intf`（Apache-2.0）、`esp_ipa`（ESPRESSIF MIT）
- 场景：摄像头采集、视频流、图像存储、USB 摄像头、ISP 图像处理、自定义传感器适配

## 安装

将本技能目录放入 Claude Code 的 skills 目录即可：

- **项目级**：复制到 `<project>/.claude/skills/esp-video-components-skill/`
- **用户级**：复制到 `~/.claude/skills/esp-video-components-skill/`

目录结构：

```
esp-video-components-skill/
├── SKILL.md              # 技能入口（含 frontmatter + 核心规则 + 陷阱 + 工作流）
├── AGENTS.md             # 工程约定（不与 SKILL.md 重复）
├── recipes/              # 11 个场景 recipe
│   ├── video_init.md
│   ├── capture_stream.md
│   ├── mipi_csi.md
│   ├── dvp_sensor.md
│   ├── spi_sensor.md
│   ├── usb_uvc_host.md
│   ├── custom_format.md
│   ├── jpeg_h264_codec.md
│   ├── image_storage.md
│   ├── simple_video_server.md
│   ├── uvc_gadget.md
│   └── isp_pipeline.md
├── resources/            # 速查文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md             # 本文件
└── CHANGELOG.md
```

安装后，在对话中提到“摄像头”、“V4L2”、“MIPI-CSI”、“esp_video”等触发词，Claude 会自动加载本技能并优先参考 recipes 与 resources。

## 许可证

技能文档按 MIT 许可发布；所引用的 esp-video-components 仓库组件分别遵循 Apache-2.0（esp_cam_sensor/esp_sccb_intf）与 ESPRESSIF MIT（esp_video/esp_ipa）。
