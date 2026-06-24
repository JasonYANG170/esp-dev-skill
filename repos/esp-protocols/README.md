# esp-protocols-skill

面向 Espressif [esp-protocols](https://github.com/espressif/esp-protocols) 仓库的 AI Skill（Claude Code / Agent skill 格式）。esp-protocols 是一组用于 ESP-IDF 的网络协议组件集合，本 Skill 聚焦其中文档完整、API 成熟的核心组件：**esp_modem**（蜂窝模组 AT + PPPoS）、**mdns**（mDNS 服务发现）、**esp_websocket_client**（WebSocket 客户端）、**eppp_link**（双 MCU PPP 组网）、**esp_dns**（DoT/DoH/TCP DNS）。

## 功能特性

- **场景化 Recipe**：覆盖 PPPoS 拨号、自定义模组、CMUX、mDNS 发布/查询、WebSocket、eppp_link、esp_dns 共 8 个真实场景，每个都给出可直接借鉴的代码与常见错误表。
- **真实 API 参考**：所有函数签名、结构体、枚举、宏均取自仓库 `components/*/include/*.h`，按组件分组。
- **配置参考**：esp_modem / mdns / eppp_link 的真实 Kconfig 选项与默认值。
- **陷阱清单**：13 条核心陷阱（含 WRONG / CORRECT 对照代码）+ 按组件组织的速查版。
- **Example 索引**：仓库内真实存在的 example 路径表，便于以 example 为起点改造。

## 安装

将本目录克隆/复制到 Claude Code 的 skills 目录即可：

- 项目级：`<项目>/.claude/skills/esp-protocols-skill/`
- 用户级：`~/.claude/skills/esp-protocols-skill/`

确保 Claude Code 能加载到 `SKILL.md`（skill 入口）。本 Skill 不修改 esp-protocols 源码，仅作参考。

## 支持范围

- **目标芯片**：ESP32 全系列（ESP32 / ESP32-S2 / ESP32-S3 / ESP32-C3 / ESP32-C6 / ESP32-P4 等，按各组件支持矩阵）
- **工具链**：ESP-IDF v5.x + CMake（`idf.py`）
- **组件版本**：esp_modem 2.0.2、mdns 1.11.2、esp_websocket_client 1.7.0（声明在各自 `idf_component.yml`）

## 目录结构

```
esp-protocols-skill/
├── SKILL.md                  # 入口：核心原则、Recipe 索引、陷阱、工作流
├── AGENTS.md                 # 工程约定：include 模式、工程结构、构建步骤、checklist
├── recipes/                  # 8 个场景 Recipe
│   ├── modem_pppos_uart.md
│   ├── modem_custom_module.md
│   ├── modem_cmux.md
│   ├── mdns_advertise.md
│   ├── mdns_query.md
│   ├── websocket_client.md
│   ├── eppp_link.md
│   └── esp_dns_secure.md
├── resources/                # 速查文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md                 # 本文件
└── CHANGELOG.md
```

## 使用建议

1. 先看 `SKILL.md` 的 "Scenario Quick Reference"，找到匹配的 Recipe。
2. 按 Recipe 的分步说明与真实代码实现，遇到 API 细节查 `resources/api_reference.md`。
3. 遇到问题先查 `resources/pitfalls.md` 与 `SKILL.md` 的 "Critical Pitfalls"。
4. 新项目优先从 `resources/example_list.md` 中最接近的真实 example 改造。

## 许可证

仓库组件源码采用 Apache-2.0（见各头文件 SPDX 标注）。本 Skill 文档同样按 Apache-2.0 提供。
