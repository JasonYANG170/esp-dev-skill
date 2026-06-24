# esp-agents-firmware-skill

面向乐鑫（Espressif）**ESP Private Agents 设备端固件 SDK**（仓库 `esp-agents-firmware`）的 AI Skill。该 SDK 用于构建与 ESP Private Agents AI 平台（<https://agents.espressif.com>）通信的设备端固件，覆盖语音对话 Agent、Matter 控制器 / Thread Border Router、本地工具、配网与板级定制。

本 Skill 让 AI Agent 在**不臆造 API** 的前提下，正确地基于该 SDK 进行固件开发、调试与定制。

## 功能特性

- **场景化 Recipes**：构建烧录、自定义板、自定义部署、Agent 初始化与事件、配网、本地工具、内置工具、Matter 控制、音频管线、串口命令共 10 个真实场景。
- **真实 API 速查**：`resources/api_reference.md` 覆盖 `esp_agent`、`app_agent`、`agent_setup`、`agent_console`、`app_audio`、`app_device`、`app_display`、Matter 控制器等模块的真实签名。
- **配置参考**：`resources/config_reference.md` 汇总所有真实 Kconfig 与板级宏。
- **陷阱汇总**：`resources/pitfalls.md` 收录 15 类高频错误与修复。
- **示例清单**：`resources/example_list.md` 索引仓库真实目录。
- 全部内容（函数名、结构体、宏、配置项、路径、代码片段）均源自仓库真实文档与源码，可查证。

## 适用范围

- 基于 `examples/voice_chat` 或 `examples/matter_controller` 创建/修改固件
- 开发新的**本地工具**（固件 + `agent_config.json` 双侧注册）
- 适配新硬件板（`examples/common/boards/`）
- 接入自定义 ESP Private Agents 部署
- 调试 Agent 连接、语音、事件、状态机、Matter 控制
- 目标芯片：ESP32-S3 系列（ESP-BOX-3 / ESP-VoCat v1.2 / M5Stack CoreS3 / +H2 Gateway）
- 工具链：ESP-IDF v5.5.2+（`release/5.5` 分支）

## 安装

将本 Skill 目录克隆/拷贝到 Claude Code 的 skills 目录即可。两种方式：

### 方式 A：项目级（仅当前工程可用）

拷贝到工程的 `.claude/skills/`：

```bash
mkdir -p .claude/skills
cp -r esp-agents-firmware-skill .claude/skills/
```

### 方式 B：用户级（所有工程可用）

拷贝到用户 skills 目录：

```bash
# Linux / macOS
cp -r esp-agents-firmware-skill ~/.claude/skills/

# Windows (Git Bash)
cp -r esp-agents-firmware-skill /c/Users/$USER/.claude/skills/
```

安装后，Claude Code 会自动识别 `esp-agents-firmware-skill`，在对话涉及 ESP Private Agents / 语音 Agent / Matter 控制器固件开发时自动加载。

## 目录结构

```
esp-agents-firmware-skill/
├── SKILL.md                  # 入口：原则、场景索引、陷阱、执行流程
├── AGENTS.md                 # 约定：包含模式、app_main 模板、构建流程、codegen 清单
├── README.md                 # 本文件
├── CHANGELOG.md              # 变更日志
├── recipes/                  # 场景化分步指南（10 个）
│   ├── build_and_flash.md
│   ├── add_custom_board.md
│   ├── custom_deployment.md
│   ├── agent_init_and_events.md
│   ├── device_setup_provisioning.md
│   ├── local_tool_register.md
│   ├── builtin_tools.md
│   ├── matter_controller_control.md
│   ├── audio_pipeline.md
│   └── console_commands.md
└── resources/                # 速查文档
    ├── api_reference.md
    ├── config_reference.md
    ├── pitfalls.md
    └── example_list.md
```

## 许可

本 Skill 内容遵循 Apache-2.0（与目标仓库 `esp-agents-firmware` 一致）。
