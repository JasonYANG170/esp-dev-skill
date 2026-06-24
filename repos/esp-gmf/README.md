# esp-gmf-skill

面向 [ESP-GMF](https://github.com/espressif/esp-gmf)（Espressif General Multimedia Framework）的 AI 开发技能（Claude Code / Agent Skill 格式）。帮助 AI 代理在 ESP32 系列芯片上正确构建音频/视频/图像流式处理流水线，所有内容**完全基于真实仓库的文档与示例**，不臆造 API。

## 这是什么

ESP-GMF 是乐鑫为 IoT 多媒体应用打造的轻量框架（RAM 占用最低约 7 KB），由 GMF-Core、Elements、Packages、GMF-Examples 四个模块组成。本技能把框架的核心概念、API、配置、陷阱浓缩为：

- **场景化配方（recipes）**：16 个真实场景（播放、录音、效果、自定义 element、状态机恢复、蓝牙音频、音视频采集、多流混音、视频显示、AI 语音前端等），每个都含真实可复制的代码。
- **API 速查（resources/api_reference.md）**：按模块分组的真实函数签名。
- **配置参考（resources/config_reference.md）**：task/IO/Kconfig/FourCC 等真实配置项。
- **陷阱汇总（resources/pitfalls.md）**：20 条高频错误与正确做法。
- **示例清单（resources/example_list.md）**：仓库真实示例路径表。

## 特性

- 中文叙事 + 英文 API/术语，便于阅读与检索
- 所有函数名、结构体、宏、配置、路径均取自 `docs/` 与源码头文件
- 配方代码改编自 `gmf_examples/basic_examples` 的真实示例
- 配套状态机、组件矩阵、控制 API 有效状态表

## 支持范围

| 维度 | 说明 |
|---|---|
| 目标芯片 | ESP32 系列 SoC（ESP32 / ESP32-S3 / ESP32-P4 / ESP32-C3 等） |
| 工具链 | ESP-IDF（`>= v5.4.3` / `>= v5.5.2` / `>= v6.0`） |
| 覆盖模块 | GMF-Core（pipeline/task/element/pool/payload/port/data_bus）、gmf_audio、gmf_io、gmf_loader、esp_audio_simple_player��esp_player、esp_capture、esp_asrc |
| 高频场景 | 播放（file/http/flash/raw）、录音编码、容器封装、音效调整、自定义 element、无缝循环、状态机/错误恢复 |

## 安装

把本目录放进 Claude Code 的 skills 目录即可。

### 方式 A：项目级（仅当前项目可用）

```bash
git clone <本仓库> /d/esp-skill/skills/esp-gmf-skill
# 软链或拷贝到项目内
mkdir -p <your_project>/.claude/skills
cp -r /d/esp-skill/skills/esp-gmf-skill <your_project>/.claude/skills/
```

### 方式 B：用户级（全局可用）

```bash
# Windows (Git Bash)
cp -r /d/esp-skill/skills/esp-gmf-skill ~/.claude/skills/

# Linux / macOS
cp -r /d/esp-skill/skills/esp-gmf-skill ~/.claude/skills/
```

安装后在涉及 ESP-GMF 的对话中，Claude 会自动加载 `SKILL.md`，并按需读取 `recipes/` 与 `resources/`。

## 目录结构

```
esp-gmf-skill/
├── SKILL.md              # 主入口：原则、状态机、陷阱、执行工作流
├── AGENTS.md             # 约定：命名、include、app_main 模板、构建流程、检查清单
├── README.md             # 本文件
├── CHANGELOG.md          # 版本记录
├── recipes/              # 16 个场景配方（.md）
│   ├── simple_player.md
│   ├── esp_player.md
│   ├── pipeline_play_embed.md
│   ├── pipeline_play_http.md
│   ├── pipeline_record.md
│   ├── pipeline_muxer.md
│   ├── audio_effects.md
│   ├── custom_element.md
│   ├── runtime_methods.md
│   ├── loop_play.md
│   └── state_error_recovery.md
└── resources/
    ├��─ api_reference.md
    ├── config_reference.md
    ├── pitfalls.md
    └── example_list.md
```

## 许可证

随 ESP-GMF 仓库：`LicenseRef-Espressif-Modified-MIT`（仅供 Espressif 产品使用）。本技能内容本身供 AI 辅助开发使用。
