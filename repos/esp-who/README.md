# esp-who-skill

ESP-WHO（乐鑫人像/视觉处理平台）AI 开发技能，面向 Claude Code / Agent Skill 格式。

本技能让 AI 智能体能够**正确地**基于 `esp-who` 仓库进行 ESP32-S3 / ESP32-P4 上的视觉 AI 固件开发：人脸检测、人脸识别、行人/猫/狗目标检测、二维码识别。所有 API、配置项、代码片段均严格来源于 `esp-who` 仓库真实的头文件、`examples/` 与 `components/`，绝不臆造。

## 功能特性

- **场景化配方（recipes/）**：覆盖工程构建、目标检测（LCD/串口）、人脸识别、二维码识别、自定义流水线、摄像头选型、任务生命周期、定制检测应用共 8 个真实场景，每个配方均含可复制代码与常见错误表。
- **完整 API 速查（resources/api_reference.md）**：按模块（任务基类、流水线、摄像头、检测、识别、二维码、应用封装、LCD、文件系统、USB）罗列真实类与方法签名。
- **配置参考（resources/config_reference.md）**：ESP-WHO 自有 Kconfig、`idf.py` 构建变量、检测模型宏、硬件能力宏、各 BSP 的关键 sdkconfig、分区表。
- **陷阱汇总（resources/pitfalls.md）**：16 条常见错误，每条附 WRONG/CORRECT 代码对比。
- **状态机与数据流（resources/state_machine.md）**：`WhoTaskBase` 生命周期、节点链数据流、检测/识别/二维码任务循环。
- **示例与组件清单（resources/example_list.md）**：全部 `examples/` 与 `components/` 路径与职责。
- **核心 SKILL.md**：12 条核心原则、配方索引、开发板/芯片支持表、模型映射表、12 条关键陷阱（WRONG/CORRECT）、执行流程、失败策略。

## 支持范围

- **芯片**：ESP32-S3、ESP32-P4
- **开发板**：ESP32-S3-EYE、ESP32-S3-Korvo-2、ESP32-P4 Function EV Board
- **ESP-IDF**：release/v5.4 或 release/v5.5
- **场景**：人脸检测、人脸识别、行人/猫/狗检测、二维码识别
- **摄像头**：S3 DVP camera、P4 MIPI-CSI（SC2336）、USB UVC
- **图形**：LVGL（LCD 模式）/ 无图形库（term 模式）

> 本技能不覆盖 esp-who 旧分支（`release/v1.1.0`）的 esp32 / esp32-s2 / 猫脸检测 / 颜色检测 API。

## 安装

将本技能目录放入 Claude Code 的 skills 目录即可，两种常见位置：

### 方式一：项目级（推荐，随工程走）

把 `esp-who-skill/` 复制到你的 ESP-WHO 工程下的 `.claude/skills/`：

```bash
mkdir -p /path/to/your-esp-who-project/.claude/skills
cp -r /d/esp-skill/skills/esp-who-skill /path/to/your-esp-who-project/.claude/skills/
```

### 方式二：用户级（全局可用）

放入用户级 skills 目录（如 `~/.claude/skills/`）：

```bash
cp -r /d/esp-skill/skills/esp-who-skill ~/.claude/skills/
```

安装后，在 Claude Code 会话中提及 ESP-WHO / 人脸识别 / 目标检测等触发词，技能会自动激活并引导按真实 API 开发。

## 目录结构

```
esp-who-skill/
├── SKILL.md                     # 主技能文件（frontmatter + 核心）
├── AGENTS.md                    # 工程约定与 checklist
├── README.md                    # 本文件
├── CHANGELOG.md                 # 变更记录
├── recipes/                     # 场景配方（8 个）
│   ├── project_setup.md
│   ├── object_detect_lcd.md
│   ├── object_detect_term.md
│   ├── custom_detect_app.md
│   ├── human_face_recognition.md
│   ├── qrcode_recognition.md
│   ├── frame_cap_pipeline.md
│   ├── camera_selection.md
│   └── task_lifecycle.md
└── resources/                   # 速查文档（5 个）
    ├── api_reference.md
    ├── config_reference.md
    ├── pitfalls.md
    ├── example_list.md
    └── state_machine.md
```

## 许可证

随 esp-who 仓库采用 ESPRESSIF MIT License（见 `SKILL.md` frontmatter）。
