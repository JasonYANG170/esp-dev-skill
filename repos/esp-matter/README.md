# esp-matter-skill

面向 Espressif **esp-matter**（基于 Matter / CHIP 协议的官方 SDK）固件开发的 AI Skill。本 Skill 为 Claude Code / Agent 形式，提供场景驱动的 recipe、真实 API 参考、配置项速查与常见陷阱，帮助 AI 正确地为 ESP32 系列芯片开发 Matter（Wi-Fi / Thread）设备、Border Router、Bridge 与 Controller。

所有内容（函数名、结构体、宏、Kconfig 选项、文件路径、代码片段）均取自 esp-matter 仓库的真实文档（`docs/`）与源码（`components/`、`examples/`），不臆造 API。

## 功能特性

- **场景 Recipe**：覆盖环境搭建、数据模型、灯具/传感器/开关/门锁/窗帘、入网、Bridge、工厂分区、OTA、Controller、设备控制台等 12 个真实场景。
- **API 速查**：`node` / `endpoint` / `cluster` / `attribute` / `command` / `event` / `identify` / `ota` / `client` 的真实签名，逐条标注头文件路径。
- **配置速查**：esp-matter 全部 Kconfig 选项（含四类 Factory Data Provider、Cluster 裁剪、多模芯片 Wi-Fi/Thread 切换）。
- **陷阱汇总**：20 条最常见的"设备不开机/不入网/不上报"错误及正/反代码对照。
- **示例索引**：仓库 `examples/` 下全部真实工程路径与一句话描述。
- **状态机**：Matter 节点从启动到入网的事件流与回调签名。

## 支持范围

- **目标芯片**：ESP32、ESP32-C2/C3/C5/C6/C61、ESP32-H2、ESP32-S3、ESP32-P4。
- **工具链**：ESP-IDF v5.5.4（`idf.py`），依赖 `connectedhomeip` 子模块（pin 到 `107f97a9813`）。
- **Matter 规范**：v1.0–v1.5（各有 release 分支），v1.6 在 `main`。
- **License**：Apache-2.0。

## 安装

将本 Skill 目录放入 Claude Code 的 skills 目录之一即可被自动发现：

- **项目级**：`<your-project>/.claude/skills/esp-matter-skill/`
- **用户级**：`~/.claude/skills/esp-matter-skill/`

或直接克隆：

```bash
git clone <this-repo> D:/esp-skill/skills/esp-matter-skill
```

## 目录结构

```
esp-matter-skill/
├── SKILL.md                      # 入口：核心原则、何时用、recipe 索引、陷阱、执行流程
├── AGENTS.md                     # 补充约定：include 模式、app_main 范式、构建流程、checklist
├── recipes/                      # 场景 recipe（中文，含真实代码与常见错误表）
│   ├── setup_and_build.md
│   ├── device_data_model.md
│   ├── custom_cluster.md
│   ├── commissioning_chiptool.md
│   ├── device_console.md
│   ├── lighting.md
│   ├── sensors.md
│   ├── switches_binding.md
│   ├── door_lock_window_covering.md
│   ├── bridge_zigbee.md
│   ├── factory_data_attestation.md
│   └── matter_ota.md
├── resources/                    # 速查文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   ├── example_list.md
│   └── state_machine.md
├── README.md
└── CHANGELOG.md
```

## 使用方式

AI Agent 在用户提及 esp-matter / Matter / 入网 / endpoint / cluster / commissioning 等关键词时，先读 `SKILL.md`，按"Scenario Quick Reference"跳到对应 recipe，需要具体 API/配置时再查 `resources/`。所有 API 与路径均可在仓库 `D:/esp-skill/espressif-repos/esp-matter/` 中核对。

## 上游参考

- 在线编程指南：https://docs.espressif.com/projects/esp-matter/
- 仓库：https://github.com/espressif/esp-matter
- Matter 博客系列：https://blog.espressif.com/matter-38ccf1d60bcd
- CSA Matter 规范：https://csa-iot.org/developer-resource/specifications-download-request/
