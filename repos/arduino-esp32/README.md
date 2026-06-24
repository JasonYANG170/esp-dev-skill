# arduino-esp32-skill

面向 **arduino-esp32**（Arduino core for ESP32）固件开发的 AI Skill。
全部内容（API 签名、宏、配置、示例路径、代码片段）均取自 arduino-esp32 仓库真实文档与源码（`docs/en/api/*.rst`、`docs/en/tutorials/*.rst`、`docs/en/migration_guides/2.x_to_3.0.rst`、`libraries/*/examples/`、`cores/esp32/`），不杜撰任何 API。

## 功能特性

- 覆盖 ESP32 / ESP32-S2 / S3 / C3 / C5 / C6 / H2 / P4（C2/C61 以 ESP-IDF 组件方式）
- 场景化 recipe：GPIO/中断、Serial、I2C、SPI、LEDC/PWM、ADC、硬件定时器、Wi-Fi STA/AP、Preferences/NVS、ESP-NOW、FreeRTOS 多任务、深睡眠
- 真实 API 速查（按模块分组）、配置/分区/Kconfig 参考、常见陷阱汇总、仓库示例索引
- 紧跟 v3.x（基于 ESP-IDF >=5.3,<6.2）：Peripheral Manager（LEDC 通道自动绑定）、统一 `Network` 库、`NetworkServer.accept()`
- 12 条关键陷阱，每条配 WRONG / CORRECT 对照代码

## 安装

将本目录克隆 / 复制到 Claude 的 skills 目录之一：

- 项目级：`<项目>/.claude/skills/arduino-esp32-skill`
- 用户级：`~/.claude/skills/arduino-esp32-skill`

随后 Claude 在涉及 ESP32 Arduino 固件开发时即可自动加载本 skill。

## 支持范围

- ✅ 创建 / 修改 / 调试 `.ino` sketch 与多文件工程
- ✅ 外设、网络、存储、低功耗、OTA、分区表
- ❌ 纯 ESP-IDF C 项目（请用 ESP-IDF skill）
- ❌ PCB 设计、AVR/STM32/ESP8266 等其他平台

## 目录结构

```
arduino-esp32-skill/
├── SKILL.md            # 主入口：原则、recipe 索引、陷阱、执行流程
├── AGENTS.md           # 约定、工具链、代码生成清单
├── recipes/            # 场景化分步指南
├── resources/          # API/配置/陷阱/示例 索引
├── README.md
└── CHANGELOG.md
```

## 许可

本 skill 文档遵循 arduino-esp32 仓库许可（LGPL-2.1）。skill 本身的组织结构可自由使用。
