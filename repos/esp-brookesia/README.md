# esp-brookesia-skill

ESP-Brookesia AI Skill —— 面向 Espressif **ESP-Brookesia**（AIoT 人机交互框架）的 Claude Code / Agent Skill。
本 Skill 让 AI Agent 在**不臆造 API** 的前提下，正确地基于 ESP-Brookesia 进行 ESP32 固件开发：系统服务
（Wi-Fi / Audio / NVS / SNTP / Device / Video / Custom）、Manager + Helper（CRTP）服务框架、HAL 板级抽象、
AI Agent（XiaoZhi / Coze / OpenAI）以及 Expression（Emote）表情动画，以及通过 MCP/Function Calling 让 LLM 调用设备能力。

> ESP-Brookesia 自 v0.7 起组件化，所有内容均基于仓库 `docs/` 与 `examples/`/源码核实，遵循 ESP-IDF 组件管理器工作流。

## 特性

- **场景化配方（recipes）**：覆盖项目搭建、服务框架通用范式、Wi-Fi/NVS/Audio/Device/SNTP/Custom 服务、AI 语音助手、Emote 表情、HAL 板级适配等 10 个真实场景，每个配方含分步代码与常见错误表。
- **API/配置速查（resources）**：按组件分组的真实函数签名、枚举、宏、Kconfig 符号、示例清单，方便快速核对。
- **踩坑清单**：服务生命周期、RAII binding/connection、参数 schema、HAL 初始化、多核锁核、Agent 状态机等高频错误，每条给出错误/正确对照。
- **执行流程**：从需求分析、配方匹配、API 核对、构建（`gen-bmgr-config` / `set-target`）到监视的完整工作流。
- **零臆造**：所有 API、路径、配置项均可在仓库 `docs/` 或源码中查到；查不到一律不写。

## 安装

将本 Skill 目录放入 Claude Code 的 skills 目录即可被自动发现。两种常见位置：

```bash
# 方式一：项目级（仅当前工程可用）
cp -r esp-brookesia-skill  <your-project>/.claude/skills/

# 方式二：用户级（所有工程可用）
cp -r esp-brookesia-skill  ~/.claude/skills/
```

也可直接在 `D:/esp-skill/skills/esp-brookesia-skill/` 原地使用。

## 目录结构

```
esp-brookesia-skill/
├── SKILL.md                 # 入口：核心原则、何时用、配方索引、板支持、踩坑、执行流程
├── AGENTS.md                # 补充约定：include 模式、命名空间别名、入口范式、构建流程、codegen 清单
├── recipes/                 # 10 个场景配方
│   ├── project_setup.md
│   ├── service_framework_basics.md
│   ├── wifi_service.md
│   ├── nvs_service.md
│   ├── audio_service.md
│   ├── device_service.md
│   ├── sntp_service.md
│   ├── custom_service.md
│   ├── agent_chatbot.md
│   ├── expression_emote.md
│   └── hal_boards.md
├── resources/               # 速查文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md
└── CHANGELOG.md
```

## 支持范围

- **框架版本**：ESP-Brookesia v0.7（master，组件化），需 ESP-IDF >= v5.5
- **目标芯片**：ESP32-S3 / ESP32-P4 / ESP32-C5 / ESP32-S31 等（视板而定）
- **受支持板**：Espressif 8 块（`esp32_s3_korvo2_v3`、`esp_box_3`、`esp_vocat_board_v1_0/v1_2`、`esp32_p4_function_ev/x`、`esp32_s31_korvo1`、`esp_sensair_shuttle`）、Waveshare 3 块、rymcu 1 块
- **资源门槛**：Flash ≥ 8MB、PSRAM ≥ 4MB（Agent 示例需 16MB / 8MB）

不在范围内：与 ESP-Brookesia 无关的纯 ESP-IDF 驱动问题、非 ESP32 平台、PCB 设计、v0.6 及以下的非组件化 LVGL screen 方案。

## 许可证

随仓库默认 Apache-2.0。本 Skill 文档仅供 AI 辅助开发参考。
