# esp-rainmaker-skill

面向 [ESP RainMaker](https://github.com/espressif/esp-rainmaker) 云 IoT 代理（Agent）固件开发的 AI Skill，适用于 Claude Code / Agent skill 格式。

ESP RainMaker 是乐鑫提供的端到端云 IoT 解决方案，让 ESP32 系列 SoC（ESP32 / S2 / S3 / C2 / C3 / C5 / C6 / H2）无需在云端做任何配置即可实现远程控制与监控。本 Skill 完全基于真实仓库的头文件（`components/esp_rainmaker/include/`）、Kconfig（`Kconfig.projbuild`）与官方示例（`examples/`）编写，确保生成的固件代码不会臆造 API。

## 功能特性

- **场景化 recipes**：覆盖节点入门、Claiming/配网、自定义/标准设备、多设备、OTA、调度与场景、标准服务、本地控制、MQTT 直发等真实场景。
- **真实 API 参���**：`resources/api_reference.md` 收录 Node/Device/Param/OTA/MQTT 等全部真实函数签名与结构体。
- **配置速查**：`resources/config_reference.md` 列出所有 `CONFIG_ESP_RMAKER_*` Kconfig 符号及默认值。
- **陷阱合集**：`SKILL.md` 与 `resources/pitfalls.md` 给出 20+ 条真实陷阱及错误/正确代码对照。
- **示例索引**：`resources/example_list.md` 列出全部官方示例路径与一行说明。

## 安装

将本���录克隆/复制到 Claude Code 的 skills 目录之一：

- **项目级**：`<project>/.claude/skills/esp-rainmaker-skill/`
- **用户级**：`~/.claude/skills/esp-rainmaker-skill/`

```bash
# 项目级
mkdir -p .claude/skills
cp -r esp-rainmaker-skill .claude/skills/

# 用户级
mkdir -p ~/.claude/skills
cp -r esp-rainmaker-skill ~/.claude/skills/
```

Claude Code 启动后会自动发现并按 `SKILL.md` 的 front matter（name/description/trigger words）匹配意图。

## 支持范围

- **目标芯片**：ESP32 / ESP32-S2 / ESP32-S3 / ESP32-C2 / ESP32-C3 / ESP32-C5 / ESP32-C6 / ESP32-H2
- **工具链**：ESP-IDF v5.1+（`idf.py`）
- **组件**：`esp_rainmaker`（`idf_component.yml` version 1.15.0）
- **场景**：C 固件开发（不含 Python host 工具、不含手机 App UI 定制）

## 目录结构

```
esp-rainmaker-skill/
├── SKILL.md                 # 主入口：核心原则、场景索引、陷阱、执行工作流
├── AGENTS.md                # 约定：命名、include、app_main 模板、构建流程、checklist
├── recipes/                 # 场景化操作指南（中文）
│   ├── getting_started.md
│   ├── claiming_and_provisioning.md
│   ├── custom_device.md
│   ├── multi_device.md
│   ├── standard_devices.md
│   ├── ota_update.md
│   ├── scheduling_scenes.md
│   ├── services.md
│   ├── local_control.md
│   └── mqtt_topics.md
├── resources/               # 速查文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md
└── CHANGELOG.md
```

## 许可

Apache-2.0（与 esp-rainmaker 仓库一致）。
