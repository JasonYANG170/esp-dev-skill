# Changelog for esp-who-skill

## 1.0.0 — 2026-06-18

首个正式版本。基于 `esp-who` 主分支（重构后，适配新版 ESP-DL，支持 ESP32-P4）。

### Added

- **SKILL.md**：YAML frontmatter（name/description/tags/license/compatibility/metadata）+ 主体（12 条核心原则、When to Use、8 个配方索引、开发板/芯片支持表、ESP-IDF 版本表、关键配置速查、检测模型与 MODEL_TIME 映射、任务状态机摘要、12 条 Critical Pitfalls 含 WRONG/CORRECT 代码、Execution Workflow、Failure Strategies、References）。
- **AGENTS.md**：项目背景、文件命名、include 模式、标准示例结构、标准 `app_main` 模式、自定义任务循环模板、构建流程、代码生成 checklist、Do Not Modify、外部资源链接。
- **recipes/**（8 个配方，均含 适用摘要 / 触发意图 / 前置条件 / 分步说明（真实可复制代码）/ 常见错误表 / 参考）：
  - `project_setup.md` — 工程构建与烧录
  - `object_detect_lcd.md` — 目标检测 + LCD
  - `object_detect_term.md` — 目标检测 + 串口（noglib）
  - `custom_detect_app.md` — 定制检测应用（override 回调）
  - `human_face_recognition.md` — 人脸识别全流程
  - `qrcode_recognition.md` — 二维码识别
  - `frame_cap_pipeline.md` — 自定义帧采集流水线
  - `camera_selection.md` — 摄像头选型与初始化
  - `task_lifecycle.md` — 任务生命周期与自定义任务
- **resources/**（5 个速查文档）：
  - `api_reference.md` — 按模块的真实 API 速查
  - `config_reference.md` — Kconfig / sdkconfig / 构建变量参考
  - `pitfalls.md` — 16 条陷阱完整说明
  - `example_list.md` — 示例与组件清单
  - `state_machine.md` — 任务状态机与数据流
- **README.md**：中文介绍、功能特性、支持范围、安装方式（项目级 / 用户级）、目录结构。
- **CHANGELOG.md**：本文件。

### Grounding

所有函数名、类名、结构体、宏、Kconfig 符号、文件路径、代码片段均来自 `esp-who` 仓库的 `components/`（`who_task`、`who_frame_cap`、`who_detect`、`who_recognition`、`who_qrcode`、`who_app/*`、`who_peripherals/*`、`who_frame_lcd_disp`）、`examples/`（`human_face_recognition`、`object_detect`、`qrcode_recognition`）、`tools/bsp_ext.py`、各 `Kconfig` 与 `sdkconfig.bsp.*`。底层 ESP-DL 模型类（`HumanFaceDetect`、`HumanFaceRecognizer` 等）仅按 ESP-WHO 中的调用签名引用，未臆造其成员。
